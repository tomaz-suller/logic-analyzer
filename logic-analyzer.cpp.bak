#include <stdio.h>

#include "hardware/clocks.h"
#include "hardware/dma.h"
#include "hardware/irq.h"
#include "hardware/pio.h"
#include "hardware/uart.h"
#include "pico/stdlib.h"

#include "capture.pio.h"

#define UART_ID uart1
#define BAUD_RATE 115200
#define UART_MSG "PING\r\n"
#define UART_INTERVAL_MS 5

#define STROBE_PIN 2
#define CAPTURE_PIN 3
#define UART_TX_PIN 4
#define UART_RX_PIN 5

#define DETECT_SM 0
#define COUNTER_SM 1

#define BUF_WORDS 16

static uint32_t data_buf[2][BUF_WORDS];
static uint32_t time_buf[2][BUF_WORDS];

static int dma_data_ping;
static int dma_data_pong;
static int dma_time_ping;
static int dma_time_pong;

static volatile bool data_ready[2];
static volatile bool time_ready[2];

static repeating_timer_t uart_timer;

static bool uart_tx_timer_cb(__unused repeating_timer_t *rt)
{
  uart_write_blocking(UART_ID, (const uint8_t *)UART_MSG, sizeof(UART_MSG) - 1);
  return true;
}

static void dma_handler(void)
{
  uint32_t pending = dma_hw->ints0;
  dma_hw->ints0 = pending;

  if(pending & (1u << dma_data_ping))
    data_ready[0] = true;
  if(pending & (1u << dma_data_pong))
    data_ready[1] = true;
  if(pending & (1u << dma_time_ping))
    time_ready[0] = true;
  if(pending & (1u << dma_time_pong))
    time_ready[1] = true;
}

static void configure_rx_channel(
  int channel,
  const volatile void *fifo,
  uint32_t *dest,
  uint dreq,
  int chain_to,
  bool start
)
{
  dma_channel_config c = dma_channel_get_default_config(channel);
  channel_config_set_transfer_data_size(&c, DMA_SIZE_32);
  channel_config_set_read_increment(&c, false);
  channel_config_set_write_increment(&c, true);
  channel_config_set_dreq(&c, dreq);
  channel_config_set_chain_to(&c, (uint)chain_to);
  dma_channel_configure(channel, &c, dest, fifo, BUF_WORDS, start);
}


// LED
#ifdef CYW43_WL_GPIO_LED_PIN
#include "pico/cyw43_arch.h"
#endif

#ifndef LED_DELAY_MS
#define LED_DELAY_MS 250
#endif

// Perform initialisation
int pico_led_init(void) {
#if defined(PICO_DEFAULT_LED_PIN)
    // A device like Pico that uses a GPIO for the LED will define PICO_DEFAULT_LED_PIN
    // so we can use normal GPIO functionality to turn the led on and off
    gpio_init(PICO_DEFAULT_LED_PIN);
    gpio_set_dir(PICO_DEFAULT_LED_PIN, GPIO_OUT);
    return PICO_OK;
#elif defined(CYW43_WL_GPIO_LED_PIN)
    // For Pico W devices we need to initialise the driver etc
    return cyw43_arch_init();
#endif
}

// Turn the led on or off
void pico_set_led(bool led_on) {
#if defined(PICO_DEFAULT_LED_PIN)
    // Just set the GPIO on or off
    gpio_put(PICO_DEFAULT_LED_PIN, led_on);
#elif defined(CYW43_WL_GPIO_LED_PIN)
    // Ask the wifi "driver" to set the GPIO on or off
    cyw43_arch_gpio_put(CYW43_WL_GPIO_LED_PIN, led_on);
#endif
}

int main()
{
  stdio_init_all();
  sleep_ms(2000);

  pico_led_init();

  uart_init(UART_ID, BAUD_RATE);
  uart_set_hw_flow(UART_ID, false, false);
  uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE);
  gpio_set_function(UART_TX_PIN, UART_FUNCSEL_NUM(UART_ID, UART_TX_PIN));
  gpio_set_function(UART_RX_PIN, UART_FUNCSEL_NUM(UART_ID, UART_RX_PIN));

  PIO pio = pio0;

  pio_gpio_init(pio, STROBE_PIN);
  pio_gpio_init(pio, CAPTURE_PIN);

  gpio_pull_up(CAPTURE_PIN);

  uint offset_detect = pio_add_program(pio, &detect_program);
  uint offset_counter = pio_add_program(pio, &counter_program);

  // Hundreds of ms per word so a 16-word half takes seconds (UART keeps up).
  // uint32_t delay_cycles = clock_get_hz(clk_sys) / 4;
  uint32_t delay_cycles = 2048;
  detect_program_init
  (
    pio, DETECT_SM, offset_detect, STROBE_PIN, CAPTURE_PIN,
    delay_cycles
  );
  counter_program_init(pio, COUNTER_SM, offset_counter, STROBE_PIN);

  dma_data_ping = dma_claim_unused_channel(true);
  dma_data_pong = dma_claim_unused_channel(true);
  dma_time_ping = dma_claim_unused_channel(true);
  dma_time_pong = dma_claim_unused_channel(true);

  configure_rx_channel
  (
    dma_data_ping, &pio->rxf[DETECT_SM], data_buf[0],
    DREQ_PIO0_RX0, dma_data_pong, true
  );
  configure_rx_channel
  (
    dma_data_pong, &pio->rxf[DETECT_SM], data_buf[1],
    DREQ_PIO0_RX0, dma_data_ping, false
  );
  configure_rx_channel
  (
    dma_time_ping, &pio->rxf[COUNTER_SM], time_buf[0],
    DREQ_PIO0_RX1, dma_time_pong, true
  );
  configure_rx_channel
  (
    dma_time_pong, &pio->rxf[COUNTER_SM], time_buf[1],
    DREQ_PIO0_RX1, dma_time_ping, false
  );

  dma_channel_set_irq0_enabled(dma_data_ping, true);
  dma_channel_set_irq0_enabled(dma_data_pong, true);
  dma_channel_set_irq0_enabled(dma_time_ping, true);
  dma_channel_set_irq0_enabled(dma_time_pong, true);
  irq_set_exclusive_handler(DMA_IRQ_0, dma_handler);
  irq_set_enabled(DMA_IRQ_0, true);


  printf("System clock %lu Hz\n", (unsigned long)clock_get_hz(clk_sys));
  printf
  (
    "SM detect=%u (offset %u) counter=%u (offset %u)\n", DETECT_SM,
    offset_detect, COUNTER_SM, offset_counter
  );
  printf
  (
    "DMA data ping=%d pong=%d  time ping=%d pong=%d\n", dma_data_ping,
    dma_data_pong, dma_time_ping, dma_time_pong
  );
  printf("Buffer size %u words\n", BUF_WORDS);
  printf("Waiting for first half\n");

  pio_enable_sm_mask_in_sync(pio, (1u << DETECT_SM) | (1u << COUNTER_SM));

  while (true) {
        pico_set_led(true);
        sleep_ms(LED_DELAY_MS);
        pico_set_led(false);
        sleep_ms(LED_DELAY_MS);
  }

  // Negative delay: fire every UART_INTERVAL_MS from the start of the last
  // callback so the gap stays regular while the capture loop dumps buffers.

  add_repeating_timer_ms(-UART_INTERVAL_MS, uart_tx_timer_cb, NULL, &uart_timer);

  while(true)
  {
    for(int half = 0; half < 2; half++)
    {
      if(data_ready[half] && time_ready[half])
      {
        // WRITE_ADDR is live and does not reload; rewind the idle half
        // so CHAIN_TO can restart it (RP2040 datasheet 2.5.1.1 / 2.5.2.2).
        if(half == 0)
        {
          dma_channel_set_write_addr(dma_data_ping, data_buf[0], false);
          dma_channel_set_write_addr(dma_time_ping, time_buf[0], false);
        }
        else
        {
          dma_channel_set_write_addr(dma_data_pong, data_buf[1], false);
          dma_channel_set_write_addr(dma_time_pong, time_buf[1], false);
        }
        for(uint i = 0; i < BUF_WORDS; i++)
        {
          printf
          (
            "%u: data=0x%08lx time=0x%08lx\n", i,
            (unsigned long)data_buf[half][i],
            (unsigned long)time_buf[half][i]
          );
        }
        data_ready[half] = false;
        time_ready[half] = false;
      }
    }
  }
}
