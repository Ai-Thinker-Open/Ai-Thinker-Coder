# BL602 UART Register-Level API Reference

**Source file:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/uart_reg.h`

**Base addresses:**
- UART0: `0x4000A000`
- UART1: `0x4000A100`

---

## Register Overview

| Offset | Register Name       | Description                                      |
|--------|---------------------|--------------------------------------------------|
| `0x00` | `utx_config`        | UART Transmitter configuration                   |
| `0x04` | `urx_config`        | UART Receiver configuration                       |
| `0x08` | `uart_bit_prd`      | Bit timing period for TX and RX                  |
| `0x0C` | `data_config`       | Data format configuration (bit inversion)         |
| `0x10` | `utx_ir_position`   | IR TX position (start/pulse)                     |
| `0x14` | `urx_ir_position`   | IR RX position (start)                           |
| `0x18` | `urx_rto_timer`     | RX timeout value                                 |
| `0x20` | `uart_int_sts`      | Interrupt status (raw)                           |
| `0x24` | `uart_int_mask`     | Interrupt mask                                   |
| `0x28` | `uart_int_clear`    | Interrupt clear (write-1-to-clear)               |
| `0x2C` | `uart_int_en`       | Interrupt enable                                 |
| `0x30` | `uart_status`       | Bus busy status                                  |
| `0x34` | `sts_urx_abr_prd`   | Auto-baud rate measurement results               |
| `0x80` | `uart_fifo_config_0` | FIFO DMA and overflow flags                     |
| `0x84` | `uart_fifo_config_1` | FIFO count and threshold                        |
| `0x88` | `uart_fifo_wdata`   | FIFO write data (TX)                             |
| `0x8C` | `uart_fifo_rdata`   | FIFO read data (RX)                             |

---

## Key Register Fields

### `utx_config` (Offset `0x00`) - TX Configuration

| Bit(s) | Field            | Description                                              | R/W  | Default |
|--------|------------------|----------------------------------------------------------|------|---------|
| `[0]`  | `cr_utx_en`      | TX enable                                                | r/w  | `0x0`   |
| `[1]`  | `cr_utx_cts_en`  | TX CTS enable                                            | r/w  | `0x0`   |
| `[2]`  | `cr_utx_frm_en`  | TX frame enable                                          | r/w  | `0x0`   |
| `[4]`  | `cr_utx_prt_en`  | Parity enable                                            | r/w  | `0x0`   |
| `[5]`  | `cr_utx_prt_sel` | Parity select (0=even, 1=odd)                           | r/w  | `0x0`   |
| `[6]`  | `cr_utx_ir_en`   | IR TX enable                                             | r/w  | `0x0`   |
| `[7]`  | `cr_utx_ir_inv`  | IR TX inversion                                          | r/w  | `0x0`   |
| `[10:8]` | `cr_utx_bit_cnt_d` | TX data bits count minus 1 (e.g. `0x7` = 8 bits)     | r/w  | `0x7`   |
| `[13:12]` | `cr_utx_bit_cnt_p` | TX stop bits count minus 1 (0=1stop, 1=1.5, 2=2stop) | r/w | `0x1` |
| `[31:16]` | `cr_utx_len`    | TX frame length in bits (0=continuous)                 | r/w  | `0x0`   |

### `urx_config` (Offset `0x04`) - RX Configuration

| Bit(s) | Field               | Description                                          | R/W  | Default |
|--------|---------------------|------------------------------------------------------|------|---------|
| `[0]`  | `cr_urx_en`         | RX enable                                            | r/w  | `0x0`   |
| `[1]`  | `cr_urx_rts_sw_mode` | RX RTS software mode                                | r/w  | `0x0`   |
| `[2]`  | `cr_urx_rts_sw_val` | RX RTS software value                               | r/w  | `0x0`   |
| `[3]`  | `cr_urx_abr_en`     | Auto-baud rate detection enable                     | r/w  | `0x0`   |
| `[4]`  | `cr_urx_prt_en`     | Parity enable                                        | r/w  | `0x0`   |
| `[5]`  | `cr_urx_prt_sel`    | Parity select (0=even, 1=odd)                       | r/w  | `0x0`   |
| `[6]`  | `cr_urx_ir_en`      | IR RX enable                                        | r/w  | `0x0`   |
| `[7]`  | `cr_urx_ir_inv`     | IR RX inversion                                     | r/w  | `0x0`   |
| `[10:8]` | `cr_urx_bit_cnt_d`  | RX data bits count minus 1                          | r/w  | `0x7`   |
| `[11]` | `cr_urx_deg_en`     | Deglitch enable                                      | r/w  | `0x0`   |
| `[15:12]` | `cr_urx_deg_cnt`  | Deglitch clock cycle count (0-15)                   | r/w  | `0x0`   |
| `[31:16]` | `cr_urx_len`      | RX frame length in bits (0=continuous)              | r/w  | `0x0`   |

### `uart_bit_prd` (Offset `0x08`) - Bit Timing Period

| Bit(s) | Field            | Description                           | R/W  | Default |
|--------|------------------|---------------------------------------|------|---------|
| `[15:0]` | `cr_utx_bit_prd` | TX bit period - 1 (clock cycles - 1) | r/w  | `0xff`  |
| `[31:16]` | `cr_urx_bit_prd` | RX bit period - 1 (clock cycles - 1) | r/w  | `0xff`  |

> Formula: `bit_period = (cr_utx_bit_prd + 1) / pclk_freq`
> For 115200 baud with 40MHz PCLK: `cr_utx_bit_prd` = `40MHz / 115200 - 1` ≈ `0xD8`

### `data_config` (Offset `0x0C`) - Data Format

| Bit(s) | Field             | Description              | R/W  | Default |
|--------|-------------------|--------------------------|------|---------|
| `[0]`  | `cr_uart_bit_inv` | UART data bit inversion   | r/w  | `0x0`   |

### `uart_int_sts` (Offset `0x20`) - Interrupt Status (Raw)

| Bit | Field          | Description                     | R/W | Default |
|-----|----------------|---------------------------------|-----|---------|
| `[0]` | `utx_end_int`  | TX end interrupt flag           | r   | `0x0`   |
| `[1]` | `urx_end_int`  | RX end interrupt flag           | r   | `0x0`   |
| `[2]` | `utx_fifo_int` | TX FIFO threshold interrupt     | r   | `0x0`   |
| `[3]` | `urx_fifo_int` | RX FIFO threshold interrupt     | r   | `0x0`   |
| `[4]` | `urx_rto_int`  | RX timeout interrupt flag       | r   | `0x0`   |
| `[5]` | `urx_pce_int`  | RX parity check error interrupt | r   | `0x0`   |
| `[6]` | `utx_fer_int`  | TX frame error interrupt        | r   | `0x0`   |
| `[7]` | `urx_fer_int`  | RX frame error interrupt        | r   | `0x0`   |

### `uart_int_mask` (Offset `0x24`) - Interrupt Mask

| Bit | Field              | Description                | R/W  | Default |
|-----|--------------------|----------------------------|------|---------|
| `[0]` | `cr_utx_end_mask`  | Mask TX end interrupt      | r/w  | `0x1`   |
| `[1]` | `cr_urx_end_mask`  | Mask RX end interrupt      | r/w  | `0x1`   |
| `[2]` | `cr_utx_fifo_mask` | Mask TX FIFO interrupt     | r/w  | `0x1`   |
| `[3]` | `cr_urx_fifo_mask` | Mask RX FIFO interrupt     | r/w  | `0x1`   |
| `[4]` | `cr_urx_rto_mask`  | Mask RX timeout interrupt  | r/w  | `0x1`   |
| `[5]` | `cr_urx_pce_mask`  | Mask RX PCE interrupt      | r/w  | `0x1`   |
| `[6]` | `cr_utx_fer_mask`  | Mask TX FER interrupt      | r/w  | `0x1`   |
| `[7]` | `cr_urx_fer_mask`  | Mask RX FER interrupt      | r/w  | `0x1`   |

### `uart_int_clear` (Offset `0x28`) - Interrupt Clear (Write-1-to-Clear)

| Bit | Field             | Description              | R/W | Default |
|-----|-------------------|--------------------------|-----|---------|
| `[0]` | `cr_utx_end_clr`  | Clear TX end flag        | w1c | `0x0`   |
| `[1]` | `cr_urx_end_clr`  | Clear RX end flag        | w1c | `0x0`   |
| `[4]` | `cr_urx_rto_clr`  | Clear RX timeout flag    | w1c | `0x0`   |
| `[5]` | `cr_urx_pce_clr`  | Clear RX PCE flag        | w1c | `0x0`   |

### `uart_int_en` (Offset `0x2C`) - Interrupt Enable

| Bit | Field            | Description              | R/W  | Default |
|-----|------------------|--------------------------|------|---------|
| `[0]` | `cr_utx_end_en`  | Enable TX end interrupt  | r/w  | `0x1`   |
| `[1]` | `cr_urx_end_en`  | Enable RX end interrupt  | r/w  | `0x1`   |
| `[2]` | `cr_utx_fifo_en` | Enable TX FIFO interrupt | r/w  | `0x1`   |
| `[3]` | `cr_urx_fifo_en` | Enable RX FIFO interrupt | r/w  | `0x1`   |
| `[4]` | `cr_urx_rto_en`  | Enable RX timeout int    | r/w  | `0x1`   |
| `[5]` | `cr_urx_pce_en`  | Enable RX PCE interrupt  | r/w  | `0x1`   |
| `[6]` | `cr_utx_fer_en`  | Enable TX FER interrupt  | r/w  | `0x1`   |
| `[7]` | `cr_urx_fer_en`  | Enable RX FER interrupt  | r/w  | `0x1`   |

### `uart_status` (Offset `0x30`) - Bus Status

| Bit | Field              | Description       | R/W | Default |
|-----|--------------------|-------------------|-----|---------|
| `[0]` | `sts_utx_bus_busy` | TX bus busy flag  | r   | `0x0`   |
| `[1]` | `sts_urx_bus_busy` | RX bus busy flag  | r   | `0x0`   |

### `uart_fifo_config_0` (Offset `0x80`) - FIFO and DMA Control

| Bit | Field                | Description                    | R/W  | Default |
|-----|----------------------|--------------------------------|------|---------|
| `[0]` | `uart_dma_tx_en`    | TX DMA enable                  | r/w  | `0x0`   |
| `[1]` | `uart_dma_rx_en`    | RX DMA enable                  | r/w  | `0x0`   |
| `[2]` | `tx_fifo_clr`       | TX FIFO clear (write 1)       | w1c  | `0x0`   |
| `[3]` | `rx_fifo_clr`       | RX FIFO clear (write 1)       | w1c  | `0x0`   |
| `[4]` | `tx_fifo_overflow`  | TX FIFO overflow flag (read)  | r    | `0x0`   |
| `[5]` | `tx_fifo_underflow` | TX FIFO underflow flag (read)  | r    | `0x0`   |
| `[6]` | `rx_fifo_overflow`  | RX FIFO overflow flag (read)   | r    | `0x0`   |
| `[7]` | `rx_fifo_underflow` | RX FIFO underflow flag (read)  | r    | `0x0`   |

### `uart_fifo_config_1` (Offset `0x84`) - FIFO Count and Threshold

| Bit(s)   | Field          | Description                      | R/W | Default |
|----------|----------------|----------------------------------|-----|---------|
| `[5:0]`  | `tx_fifo_cnt`  | TX FIFO count (0-32)             | r   | `0x20`  |
| `[13:8]` | `rx_fifo_cnt`  | RX FIFO count (0-32)             | r   | `0x0`   |
| `[20:16]` | `tx_fifo_th`  | TX FIFO threshold                | r/w | `0x0`   |
| `[28:24]` | `rx_fifo_th`  | RX FIFO threshold                | r/w | `0x0`   |

### `uart_fifo_wdata` (Offset `0x88`) - TX FIFO Write

| Bit(s)  | Field             | Description       | R/W | Default |
|---------|-------------------|-------------------|-----|---------|
| `[7:0]` | `uart_fifo_wdata` | TX FIFO write data | w   | undefined |

### `uart_fifo_rdata` (Offset `0x8C`) - RX FIFO Read

| Bit(s)  | Field             | Description       | R/W | Default |
|---------|-------------------|-------------------|-----|---------|
| `[7:0]` | `uart_fifo_rdata` | RX FIFO read data  | r   | `0x0`   |

---

## Register-Level Programming Example

### Initialize UART0 at 115200 Baud, 8N1, Polling Mode

```c
#include "bl602.h"
#include "uart_reg.h"

#define UART0_BASE (0x4000A000)
#define PCLK_FREQ  40000000UL  /* 40 MHz */

static volatile uart_reg_t *uart0 = (uart_reg_t *)UART0_BASE;

void uart0_init(void)
{
    uint32_t bit_prd;

    /* Calculate bit period: bit_prd = PCLK / baud - 1 */
    bit_prd = PCLK_FREQ / 115200UL - 1;

    /* Disable TX and RX during configuration */
    uart0->utx_config.WORD = 0;
    uart0->urx_config.WORD = 0;

    /* Configure bit timing: 8 data bits, 1 stop bit */
    uart0->uart_bit_prd.WORD = (bit_prd << 16) | bit_prd;

    /* Configure TX: 8-bit mode, no parity, no IR, no frame */
    uart0->utx_config.BF.cr_utx_en = 1;
    uart0->utx_config.BF.cr_utx_bit_cnt_d = 7;  /* 8 - 1 = 7 */
    uart0->utx_config.BF.cr_utx_bit_cnt_p = 0; /* 1 stop bit */
    uart0->utx_config.BF.cr_utx_prt_en = 0;     /* no parity */

    /* Configure RX: 8-bit mode, no parity, no IR */
    uart0->urx_config.BF.cr_urx_en = 1;
    uart0->urx_config.BF.cr_urx_bit_cnt_d = 7;  /* 8 - 1 = 7 */
    uart0->urx_config.BF.cr_urx_bit_cnt_p = 0; /* 1 stop bit */
    uart0->urx_config.BF.cr_urx_prt_en = 0;     /* no parity */
}
```

### Send a Byte (Polling)

```c
void uart0_putc(uint8_t ch)
{
    /* Wait until TX FIFO has space */
    while (uart0->uart_fifo_config_1.BF.tx_fifo_cnt == 0)
        ;

    /* Write byte to TX FIFO */
    uart0->uart_fifo_wdata.BF.uart_fifo_wdata = ch;
}
```

### Receive a Byte (Polling)

```c
uint8_t uart0_getc(void)
{
    /* Wait until RX FIFO has data */
    while (uart0->uart_fifo_config_1.BF.rx_fifo_cnt == 0)
        ;

    /* Read byte from RX FIFO */
    return (uint8_t)uart0->uart_fifo_rdata.BF.uart_fifo_rdata;
}
```

### Send a String

```c
void uart0_puts(const char *s)
{
    while (*s) {
        if (*s == '\n')
            uart0_putc('\r');
        uart0_putc(*s++);
    }
}
```

### Configure UART0 with RX Timeout Interrupt

```c
void uart0_init_it(void)
{
    uint32_t bit_prd = PCLK_FREQ / 115200UL - 1;

    /* Disable TX/RX for configuration */
    uart0->utx_config.WORD = 0;
    uart0->urx_config.WORD = 0;

    /* Bit timing */
    uart0->uart_bit_prd.WORD = (bit_prd << 16) | bit_prd;

    /* TX config */
    uart0->utx_config.BF.cr_utx_en = 1;
    uart0->utx_config.BF.cr_utx_bit_cnt_d = 7;
    uart0->utx_config.BF.cr_utx_bit_cnt_p = 0;
    uart0->utx_config.BF.cr_utx_prt_en = 0;

    /* RX config with timeout */
    uart0->urx_config.BF.cr_urx_en = 1;
    uart0->urx_config.BF.cr_urx_bit_cnt_d = 7;
    uart0->urx_config.BF.cr_urx_bit_cnt_p = 0;
    uart0->urx_config.BF.cr_urx_prt_en = 0;

    /* Set RX timeout: RTO_VALUE in bit periods */
    uart0->urx_rto_timer.BF.cr_urx_rto_value = 20; /* ~20 bit periods */

    /* Clear all pending interrupts */
    uart0->uart_int_clear.WORD = 0xFF;

    /* Enable RX end, RX FIFO, and RX timeout interrupts */
    uart0->uart_int_en.BF.cr_urx_end_en = 1;
    uart0->uart_int_en.BF.cr_urx_fifo_en = 1;
    uart0->uart_int_en.BF.cr_urx_rto_en = 1;

    /* Unmask the corresponding interrupts */
    uart0->uart_int_mask.BF.cr_urx_end_mask = 0;
    uart0->uart_int_mask.BF.cr_urx_fifo_mask = 0;
    uart0->uart_int_mask.BF.cr_urx_rto_mask = 0;
}
```

### Check RX Timeout and Parity Error in ISR

```c
void UART0_IRQHandler(void)
{
    uint32_t status = uart0->uart_int_sts.WORD;

    if (status & (1 << 4)) {  /* urx_rto_int */
        /* RX timeout: incomplete frame received */
        uart0->uart_int_clear.BF.cr_urx_rto_clr = 1;
    }

    if (status & (1 << 5)) {  /* urx_pce_int */
        /* Parity check error */
        uart0->uart_int_clear.BF.cr_urx_pce_clr = 1;
    }

    if (status & (1 << 1)) {  /* urx_end_int */
        /* RX complete */
        uint8_t ch = (uint8_t)uart0->uart_fifo_rdata.BF.uart_fifo_rdata;
        uart0->uart_int_clear.BF.cr_urx_end_clr = 1;
        /* process ch... */
    }
}
```

### Use Auto-Baud Rate Detection (0x55 Pattern)

```c
void uart0_autobaud(void)
{
    /* Disable RX/TX */
    uart0->utx_config.WORD = 0;
    uart0->urx_config.WORD = 0;

    /* Configure RX for auto-baud */
    uart0->urx_config.BF.cr_urx_en = 1;
    uart0->urx_config.BF.cr_urx_bit_cnt_d = 7;
    uart0->urx_config.BF.cr_urx_abr_en = 1;  /* Enable ABR */

    /* Clear any pending status */
    uart0->uart_int_clear.WORD = 0xFF;

    /* Wait for auto-baud measurement to complete (polling) */
    while (uart0->sts_urx_abr_prd.BF.sts_urx_abr_prd_0x55 == 0)
        ;

    /* Read measured bit period */
    uint32_t abr_prd = uart0->sts_urx_abr_prd.BF.sts_urx_abr_prd_0x55;

    /* Configure TX with measured period */
    uart0->uart_bit_prd.BF.cr_utx_bit_prd = abr_prd;

    /* Now configure TX normally */
    uart0->utx_config.BF.cr_utx_en = 1;
    uart0->utx_config.BF.cr_utx_bit_cnt_d = 7;
    uart0->utx_config.BF.cr_utx_bit_cnt_p = 0;
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_uart.h`

The HOSAL UART layer provides a high-level abstraction above the register interface.

### Initialization

```c
int hosal_uart_init(hosal_uart_dev_t *uart);
int hosal_uart_init_only_tx(hosal_uart_dev_t *uart);
```

### Send / Receive (Polling)

```c
int hosal_uart_send(hosal_uart_dev_t *uart, const void *txbuf, uint32_t size);
int hosal_uart_receive(hosal_uart_dev_t *uart, void *data, uint32_t expect_size);
```

### IO Control

```c
int hosal_uart_ioctl(hosal_uart_dev_t *uart, int ctl, void *p_arg);
```

Supported `ctl` values:
- `HOSAL_UART_BAUD_SET` / `HOSAL_UART_BAUD_GET` - baud rate
- `HOSAL_UART_DATA_WIDTH_SET` / `HOSAL_UART_DATA_WIDTH_GET` - data width (`HOSAL_DATA_WIDTH_5BIT` ... `HOSAL_DATA_WIDTH_9BIT`)
- `HOSAL_UART_STOP_BITS_SET` / `HOSAL_UART_STOP_BITS_GET` - stop bits (`HOSAL_STOP_BITS_1`, `HOSAL_STOP_BITS_1_5`, `HOSAL_STOP_BITS_2`)
- `HOSAL_UART_PARITY_SET` / `HOSAL_UART_PARITY_GET` - parity (`HOSAL_NO_PARITY`, `HOSAL_ODD_PARITY`, `HOSAL_EVEN_PARITY`)
- `HOSAL_UART_MODE_SET` / `HOSAL_UART_MODE_GET` - mode (`HOSAL_UART_MODE_POLL`, `HOSAL_UART_MODE_INT_TX`, `HOSAL_UART_MODE_INT_RX`, `HOSAL_UART_MODE_INT`)
- `HOSAL_UART_FLOWMODE_SET` / `HOSAL_UART_FLOWSTAT_GET` - flow control
- `HOSAL_UART_FREE_TXFIFO_GET` / `HOSAL_UART_FREE_RXFIFO_GET` - FIFO status
- `HOSAL_UART_FLUSH` - wait for TX complete
- `HOSAL_UART_DMA_TX_START` / `HOSAL_UART_DMA_RX_START` - DMA transfer

### Callbacks

```c
int hosal_uart_callback_set(hosal_uart_dev_t *uart, int callback_type,
                            hosal_uart_callback_t pfn_callback, void *arg);
```

Callback types:
- `HOSAL_UART_TX_CALLBACK` - TX idle interrupt
- `HOSAL_UART_RX_CALLBACK` - RX complete interrupt
- `HOSAL_UART_TX_DMA_CALLBACK` - TX DMA complete
- `HOSAL_UART_RX_DMA_CALLBACK` - RX DMA complete

### Auto-Baud

```c
int hosal_uart_abr_get(hosal_uart_dev_t *uart, uint8_t mode);
```

- `mode = HOSAL_UART_AUTOBAUD_0X55` - detect using 0x55 pattern
- `mode = HOSAL_UART_AUTOBAUD_STARTBIT` - detect using start bit

### Example using HOSAL

```c
#include "hosal_uart.h"

hosal_uart_dev_t uart0;

void uart0_example(void)
{
    /* Configure UART0 on pins TX=11, RX=14 at 115200 baud */
    HOSAL_UART_CFG_DECL(uart0_cfg, 0, 11, 14, 115200);

    uart0.config = uart0_cfg;

    hosal_uart_init(&uart0);

    /* Send data */
    const char *msg = "Hello\r\n";
    hosal_uart_send(&uart0, msg, strlen(msg));

    /* Receive data */
    uint8_t buf[64];
    hosal_uart_receive(&uart0, buf, sizeof(buf));

    hosal_uart_finalize(&uart0);
}
```
