# BL602 SPI Register-Level API Reference

**Source file:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/spi_reg.h`

**Base address:** SPI: `0x4000A200`

---

## Register Overview

| Offset | Register Name         | Description                              |
|--------|-----------------------|------------------------------------------|
| `0x00` | `spi_config`          | SPI mode, frame size, clock polarity/phase |
| `0x04` | `spi_int_sts`         | Interrupt status, mask, clear, enable     |
| `0x08` | `spi_bus_busy`        | SPI bus busy flag                        |
| `0x10` | `spi_prd_0`           | SPI clock period phases 0 and 1 (data)    |
| `0x14` | `spi_prd_1`           | SPI clock idle period                    |
| `0x18` | `spi_rxd_ignr`        | RXD ignore count (for mixed-width frames) |
| `0x1C` | `spi_sto_value`       | Slave timeout value                      |
| `0x80` | `spi_fifo_config_0`   | FIFO DMA and overflow flags              |
| `0x84` | `spi_fifo_config_1`   | FIFO count and threshold                 |
| `0x88` | `spi_fifo_wdata`      | FIFO write data                          |
| `0x8C` | `spi_fifo_rdata`      | FIFO read data                           |

---

## Key Register Fields

### `spi_config` (Offset `0x00`) - SPI Configuration

| Bit(s) | Field               | Description                                        | R/W  | Default |
|--------|---------------------|----------------------------------------------------|------|---------|
| `[0]`  | `cr_spi_m_en`       | SPI master enable                                  | r/w  | `0x0`   |
| `[1]`  | `cr_spi_s_en`       | SPI slave enable                                   | r/w  | `0x0`   |
| `[3:2]` | `cr_spi_frame_size` | Frame size: 0=4bit, 1=8bit, 2=16bit, 3=32bit      | r/w  | `0x0`   |
| `[4]`  | `cr_spi_sclk_pol`   | SCLK polarity (0=idle low, 1=idle high)           | r/w  | `0x0`   |
| `[5]`  | `cr_spi_sclk_ph`    | SCLK phase (0=first edge, 1=second edge)          | r/w  | `0x0`   |
| `[6]`  | `cr_spi_bit_inv`    | Bit order inversion (0=MSB first, 1=LSB first)   | r/w  | `0x0`   |
| `[7]`  | `cr_spi_byte_inv`   | Byte order inversion                               | r/w  | `0x0`   |
| `[8]`  | `cr_spi_rxd_ignr_en`| RXD ignore enable                                  | r/w  | `0x0`   |
| `[9]`  | `cr_spi_m_cont_en`  | Master continuous transfer enable                 | r/w  | `0x0`   |
| `[11]` | `cr_spi_deg_en`     | Deglitch enable                                    | r/w  | `0x0`   |
| `[15:12]` | `cr_spi_deg_cnt` | Deglitch clock cycle count (0-15)                | r/w  | `0x0`   |

> **Clock polarity and phase (SCLK_POL, SCLK_PH):**
> - `0, 0`: CPOL=0, CPHA=0 — idle low, sample on first (rising) edge
> - `0, 1`: CPOL=0, CPHA=1 — idle low, sample on second (falling) edge
> - `1, 0`: CPOL=1, CPHA=1 — idle high, sample on first (falling) edge
> - `1, 1`: CPOL=1, CPHA=1 — idle high, sample on second (rising) edge

### `spi_int_sts` (Offset `0x04`) - Interrupt Status, Mask, Clear, Enable

This register combines four functions in one 32-bit word:

| Bit(s)  | Field               | Description                          | R/W  | Default |
|---------|---------------------|--------------------------------------|------|---------|
| **Status (read-clear)** |||||
| `[0]`   | `spi_end_int`       | Transfer end interrupt flag          | r    | `0x0`   |
| `[1]`   | `spi_txf_int`       | TX FIFO threshold interrupt flag     | r    | `0x0`   |
| `[2]`   | `spi_rxf_int`       | RX FIFO threshold interrupt flag     | r    | `0x0`   |
| `[3]`   | `spi_sto_int`       | Slave timeout interrupt flag          | r    | `0x0`   |
| `[4]`   | `spi_txu_int`       | TX underflow interrupt flag          | r    | `0x0`   |
| `[5]`   | `spi_fer_int`       | Frame error interrupt flag            | r    | `0x0`   |
| **Mask (r/w)** |||||
| `[8]`   | `cr_spi_end_mask`   | Mask end interrupt                   | r/w  | `0x1`   |
| `[9]`   | `cr_spi_txf_mask`   | Mask TX FIFO interrupt                | r/w  | `0x1`   |
| `[10]`  | `cr_spi_rxf_mask`   | Mask RX FIFO interrupt                | r/w  | `0x1`   |
| `[11]`  | `cr_spi_sto_mask`   | Mask slave timeout interrupt         | r/w  | `0x1`   |
| `[12]`  | `cr_spi_txu_mask`   | Mask TX underflow interrupt          | r/w  | `0x1`   |
| `[13]`  | `cr_spi_fer_mask`   | Mask frame error interrupt            | r/w  | `0x1`   |
| **Clear (write-1-to-clear)** |||||
| `[16]`  | `cr_spi_end_clr`    | Clear end interrupt flag              | w1c  | `0x0`   |
| `[19]`  | `cr_spi_sto_clr`    | Clear slave timeout flag              | w1c  | `0x0`   |
| `[20]`  | `cr_spi_txu_clr`    | Clear TX underflow flag               | w1c  | `0x0`   |
| **Enable (r/w)** |||||
| `[24]`  | `cr_spi_end_en`     | Enable end interrupt                  | r/w  | `0x1`   |
| `[25]`  | `cr_spi_txf_en`     | Enable TX FIFO interrupt              | r/w  | `0x1`   |
| `[26]`  | `cr_spi_rxf_en`     | Enable RX FIFO interrupt              | r/w  | `0x1`   |
| `[27]`  | `cr_spi_sto_en`     | Enable slave timeout interrupt        | r/w  | `0x1`   |
| `[28]`  | `cr_spi_txu_en`     | Enable TX underflow interrupt        | r/w  | `0x1`   |
| `[29]`  | `cr_spi_fer_en`     | Enable frame error interrupt          | r/w  | `0x1`   |

### `spi_bus_busy` (Offset `0x08`) - Bus Busy Status

| Bit | Field              | Description       | R/W | Default |
|-----|--------------------|-------------------|-----|---------|
| `[0]` | `sts_spi_bus_busy` | SPI bus busy flag | r   | `0x0`   |

### `spi_prd_0` (Offset `0x10`) - Clock Timing Phase 0 and 1

| Bit(s)   | Field            | Description                                    | R/W  | Default |
|----------|------------------|------------------------------------------------|------|---------|
| `[7:0]`  | `cr_spi_prd_s`   | SCLK start delay (clock cycles before first edge) | r/w | `0xf` |
| `[15:8]` | `cr_spi_prd_p`   | SCLK period (clock cycles between edges)      | r/w  | `0xf`  |
| `[23:16]` | `cr_spi_prd_d_ph_0` | Data setup/hold time for phase 0             | r/w  | `0xf`  |
| `[31:24]` | `cr_spi_prd_d_ph_1` | Data setup/hold time for phase 1             | r/w  | `0xf`  |

> The effective SCLK period = `cr_spi_prd_s + cr_spi_prd_p + 2` PCLK cycles.

### `spi_prd_1` (Offset `0x14`) - Idle and Inter-Frame Timing

| Bit(s)  | Field           | Description                      | R/W  | Default |
|---------|-----------------|----------------------------------|------|---------|
| `[7:0]` | `cr_spi_prd_i`  | Inter-frame idle cycles          | r/w  | `0xf`   |

### `spi_rxd_ignr` (Offset `0x18`) - RXD Ignore Count

Used for mixed-width transfers where the TX and RX word sizes differ.

| Bit(s)   | Field               | Description                | R/W  | Default |
|----------|---------------------|----------------------------|------|---------|
| `[4:0]`  | `cr_spi_rxd_ignr_p` | Ignore count for phase 0  | r/w  | `0x0`   |
| `[20:16]` | `cr_spi_rxd_ignr_s` | Ignore count for phase 1  | r/w  | `0x0`   |

### `spi_sto_value` (Offset `0x1C`) - Slave Timeout Value

| Bit(s)  | Field              | Description                           | R/W  | Default |
|---------|--------------------|---------------------------------------|------|---------|
| `[11:0]` | `cr_spi_sto_value` | Slave timeout period in PCLK cycles  | r/w  | `0xfff` |

### `spi_fifo_config_0` (Offset `0x80`) - FIFO and DMA Control

| Bit | Field                | Description                    | R/W  | Default |
|-----|----------------------|--------------------------------|------|---------|
| `[0]` | `spi_dma_tx_en`     | TX DMA enable                  | r/w  | `0x0`   |
| `[1]` | `spi_dma_rx_en`      | RX DMA enable                  | r/w  | `0x0`   |
| `[2]` | `tx_fifo_clr`        | TX FIFO clear (write 1)        | w1c  | `0x0`   |
| `[3]` | `rx_fifo_clr`        | RX FIFO clear (write 1)        | w1c  | `0x0`   |
| `[4]` | `tx_fifo_overflow`   | TX FIFO overflow flag (read)   | r    | `0x0`   |
| `[5]` | `tx_fifo_underflow`  | TX FIFO underflow flag (read)  | r    | `0x0`   |
| `[6]` | `rx_fifo_overflow`   | RX FIFO overflow flag (read)   | r    | `0x0`   |
| `[7]` | `rx_fifo_underflow`  | RX FIFO underflow flag (read)  | r    | `0x0`   |

### `spi_fifo_config_1` (Offset `0x84`) - FIFO Count and Threshold

| Bit(s)   | Field          | Description             | R/W | Default |
|----------|----------------|-------------------------|-----|---------|
| `[2:0]`  | `tx_fifo_cnt`  | TX FIFO count (0-4)    | r   | `0x4`   |
| `[10:8]` | `rx_fifo_cnt`  | RX FIFO count (0-4)    | r   | `0x0`   |
| `[17:16]` | `tx_fifo_th`  | TX FIFO threshold       | r/w | `0x0`   |
| `[25:24]` | `rx_fifo_th`  | RX FIFO threshold       | r/w | `0x0`   |

### `spi_fifo_wdata` (Offset `0x88`) - FIFO Write Data

| Bit(s)  | Field           | Description             | R/W | Default |
|---------|-----------------|-------------------------|-----|---------|
| `[31:0]` | `spi_fifo_wdata` | TX FIFO write data (32-bit) | w | undefined |

### `spi_fifo_rdata` (Offset `0x8C`) - FIFO Read Data

| Bit(s)  | Field           | Description             | R/W | Default |
|---------|-----------------|-------------------------|-----|---------|
| `[31:0]` | `spi_fifo_rdata` | RX FIFO read data (32-bit) | r | `0x0`   |

---

## Register-Level Programming Example

### Initialize SPI as Master, Mode 0, 1MHz

```c
#include "bl602.h"
#include "spi_reg.h"

#define SPI_BASE   (0x4000A200)
#define PCLK_FREQ  40000000UL  /* 40 MHz */

static volatile spi_reg_t *spi = (spi_reg_t *)SPI_BASE;

void spi_master_init(void)
{
    /* Disable SPI during configuration */
    spi->spi_config.WORD = 0;

    /* Configure clock: Mode 0 (CPOL=0, CPHA=0), MSB first */
    spi->spi_config.BF.cr_spi_sclk_pol = 0;  /* idle low */
    spi->spi_config.BF.cr_spi_sclk_ph  = 0;  /* sample on first edge */
    spi->spi_config.BF.cr_spi_bit_inv  = 0;  /* MSB first */
    spi->spi_config.BF.cr_spi_byte_inv = 0;

    /* Frame size: 8 bits */
    spi->spi_config.BF.cr_spi_frame_size = 1;  /* 0=4bit, 1=8bit, 2=16bit, 3=32bit */

    /* Clock timing: PCLK / (2 * (prd_s + prd_p + 1))
     * For 1MHz from 40MHz: div = 40MHz / 1MHz = 40
     * prd_s = 0, prd_p = 19 gives: 40MHz / (2 * 20) = 1MHz */
    spi->spi_prd_0.BF.cr_spi_prd_s     = 0;
    spi->spi_prd_0.BF.cr_spi_prd_p     = 19;
    spi->spi_prd_0.BF.cr_spi_prd_d_ph_0 = 0;
    spi->spi_prd_0.BF.cr_spi_prd_d_ph_1 = 0;
    spi->spi_prd_1.BF.cr_spi_prd_i      = 0;

    /* Clear FIFOs */
    spi->spi_fifo_config_0.BF.tx_fifo_clr = 1;
    spi->spi_fifo_config_0.BF.rx_fifo_clr = 1;

    /* Enable master */
    spi->spi_config.BF.cr_spi_m_en = 1;
}
```

### Initialize SPI as Slave, Mode 0, 8-bit

```c
void spi_slave_init(void)
{
    /* Disable SPI during configuration */
    spi->spi_config.WORD = 0;

    /* Configure as slave */
    spi->spi_config.BF.cr_spi_sclk_pol = 0;  /* match master's idle state */
    spi->spi_config.BF.cr_spi_sclk_ph  = 0;  /* match master's phase */
    spi->spi_config.BF.cr_spi_bit_inv  = 0;
    spi->spi_config.BF.cr_spi_frame_size = 1;  /* 8-bit */

    /* Clear FIFOs */
    spi->spi_fifo_config_0.BF.tx_fifo_clr = 1;
    spi->spi_fifo_config_0.BF.rx_fifo_clr = 1;

    /* Enable slave */
    spi->spi_config.BF.cr_spi_s_en = 1;
}
```

### Exchange Data (Full-Duplex Polling)

```c
/* Send and receive n bytes. For each byte sent, one byte is received. */
void spi_transfer(uint8_t *tx_data, uint8_t *rx_data, uint32_t len)
{
    uint32_t i;

    for (i = 0; i < len; i++) {
        /* Wait for TX FIFO to have space */
        while (spi->spi_fifo_config_1.BF.tx_fifo_cnt >= 4)
            ;

        /* Write to TX FIFO */
        spi->spi_fifo_wdata.WORD = tx_data ? tx_data[i] : 0;

        /* Wait for RX FIFO to have data */
        while (spi->spi_fifo_config_1.BF.rx_fifo_cnt == 0)
            ;

        /* Read from RX FIFO */
        if (rx_data)
            rx_data[i] = (uint8_t)spi->spi_fifo_rdata.WORD;
        else
            (void)spi->spi_fifo_rdata.WORD;
    }
}
```

### Send Data Only

```c
void spi_send(const uint8_t *data, uint32_t len)
{
    uint32_t i;

    for (i = 0; i < len; i++) {
        while (spi->spi_fifo_config_1.BF.tx_fifo_cnt >= 4)
            ;
        spi->spi_fifo_wdata.WORD = data[i];
    }
}
```

### Receive Data Only

```c
void spi_recv(uint8_t *data, uint32_t len)
{
    uint32_t i;

    /* With no TX data, the SPI master drives SCLK.
     * We send 0xFF (or 0x00) for each byte to generate clocks. */
    for (i = 0; i < len; i++) {
        while (spi->spi_fifo_config_1.BF.tx_fifo_cnt >= 4)
            ;
        spi->spi_fifo_wdata.WORD = 0xFF;  /* drive SCLK */

        while (spi->spi_fifo_config_1.BF.rx_fifo_cnt == 0)
            ;
        data[i] = (uint8_t)spi->spi_fifo_rdata.WORD;
    }
}
```

### Use Interrupts for RX/TX

```c
void spi_init_it(void)
{
    /* Disable SPI during configuration */
    spi->spi_config.WORD = 0;

    /* Configure for master, mode 0, 8-bit */
    spi->spi_config.BF.cr_spi_sclk_pol = 0;
    spi->spi_config.BF.cr_spi_sclk_ph  = 0;
    spi->spi_config.BF.cr_spi_frame_size = 1;

    /* Clock timing: 1MHz */
    spi->spi_prd_0.BF.cr_spi_prd_s = 0;
    spi->spi_prd_0.BF.cr_spi_prd_p = 19;
    spi->spi_prd_0.BF.cr_spi_prd_d_ph_0 = 0;
    spi->spi_prd_0.BF.cr_spi_prd_d_ph_1 = 0;
    spi->spi_prd_1.BF.cr_spi_prd_i = 0;

    /* Clear FIFOs */
    spi->spi_fifo_config_0.BF.tx_fifo_clr = 1;
    spi->spi_fifo_config_0.BF.rx_fifo_clr = 1;

    /* Clear all pending interrupts */
    spi->spi_int_sts.BF.cr_spi_end_clr = 1;
    spi->spi_int_sts.BF.cr_spi_sto_clr = 1;
    spi->spi_int_sts.BF.cr_spi_txu_clr = 1;

    /* Enable end, RX FIFO, and RX threshold interrupts */
    spi->spi_int_sts.BF.cr_spi_end_en = 1;
    spi->spi_int_sts.BF.cr_spi_rxf_en = 1;

    /* Unmask end and RXF interrupts */
    spi->spi_int_sts.BF.cr_spi_end_mask = 0;
    spi->spi_int_sts.BF.cr_spi_rxf_mask = 0;

    /* Enable master */
    spi->spi_config.BF.cr_spi_m_en = 1;
}

void SPI_IRQHandler(void)
{
    uint32_t sts = spi->spi_int_sts.WORD;

    /* End interrupt: transfer complete */
    if (sts & (1 << 0)) {
        spi->spi_int_sts.BF.cr_spi_end_clr = 1;
        /* handle transfer complete... */
    }

    /* RX FIFO interrupt: data available */
    if (sts & (1 << 2)) {
        uint32_t rxcnt = spi->spi_fifo_config_1.BF.rx_fifo_cnt;
        while (rxcnt--) {
            uint8_t byte = (uint8_t)spi->spi_fifo_rdata.WORD;
            /* process byte... */
        }
    }

    /* Frame error */
    if (sts & (1 << 5)) {
        spi->spi_int_sts.BF.cr_spi_end_clr = 1;
    }
}
```

### Configure for 10MHz, Mode 3 (CPOL=1, CPHA=1), 16-bit Frame

```c
void spi_master_init_fast(void)
{
    spi->spi_config.WORD = 0;

    /* Mode 3: idle high, sample on second edge */
    spi->spi_config.BF.cr_spi_sclk_pol = 1;
    spi->spi_config.BF.cr_spi_sclk_ph  = 1;
    spi->spi_config.BF.cr_spi_bit_inv  = 0;

    /* 16-bit frame */
    spi->spi_config.BF.cr_spi_frame_size = 2;

    /* 10MHz from 40MHz: div = 4
     * prd_s = 0, prd_p = 1: 40MHz / (2 * 2) = 10MHz */
    spi->spi_prd_0.BF.cr_spi_prd_s     = 0;
    spi->spi_prd_0.BF.cr_spi_prd_p     = 1;
    spi->spi_prd_0.BF.cr_spi_prd_d_ph_0 = 0;
    spi->spi_prd_0.BF.cr_spi_prd_d_ph_1 = 0;
    spi->spi_prd_1.BF.cr_spi_prd_i      = 0;

    /* Clear FIFOs */
    spi->spi_fifo_config_0.BF.tx_fifo_clr = 1;
    spi->spi_fifo_config_0.BF.rx_fifo_clr = 1;

    spi->spi_config.BF.cr_spi_m_en = 1;
}
```

### Continuous Master Transfer

```c
/* Enable continuous transfer mode (CS stays low across frames) */
void spi_master_continuous(void)
{
    spi->spi_config.BF.cr_spi_m_cont_en = 1;
}

/* Transmit 16-bit words continuously */
void spi_send_16bit_cont(const uint16_t *data, uint32_t len)
{
    while (len--) {
        while (spi->spi_fifo_config_1.BF.tx_fifo_cnt >= 4)
            ;
        spi->spi_fifo_wdata.WORD = *data++;
    }
}
```

### Handle Slave Timeout

```c
/* Configure slave timeout: ~1ms at 40MHz (0x9C40 = 40000 cycles) */
spi->spi_sto_value.BF.cr_spi_sto_value = 40000;

/* Enable slave timeout interrupt */
spi->spi_int_sts.BF.cr_spi_sto_en   = 1;
spi->spi_int_sts.BF.cr_spi_sto_mask = 0;

/* In ISR */
if (spi->spi_int_sts.BF.spi_sto_int) {
    spi->spi_int_sts.BF.cr_spi_sto_clr = 1;
    /* handle timeout... */
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_spi.h`

The HOSAL SPI layer provides a higher-level abstraction above the raw registers.

### Initialization

```c
int hosal_spi_init(hosal_spi_dev_t *spi);
```

### Send / Receive

```c
int hosal_spi_send(hosal_spi_dev_t *spi, const uint8_t *data, uint16_t size, uint32_t timeout);
int hosal_spi_recv(hosal_spi_dev_t *spi, uint8_t *data, uint16_t size, uint32_t timeout);
int hosal_spi_send_recv(hosal_spi_dev_t *spi, uint8_t *tx_data, uint8_t *rx_data,
                        uint16_t size, uint32_t timeout);
```

- `timeout` in milliseconds; use `HAL_WAIT_FOREVER` (`0xFFFFFFFF`) to block indefinitely.

### Interrupt Callback

```c
int hosal_spi_irq_callback_set(hosal_spi_dev_t *spi, hosal_spi_irq_t pfn, void *p_arg);
```

### Software CS Control (Master Only)

```c
int hosal_spi_set_cs(uint8_t pin, uint8_t value);
```

### Finalize

```c
int hosal_spi_finalize(hosal_spi_dev_t *spi);
```

### Configuration Struct

```c
typedef struct {
    uint8_t mode;         /* HOSAL_SPI_MODE_MASTER (0) or HOSAL_SPI_MODE_SLAVE (1) */
    uint8_t dma_enable;   /* 0 = polling/FIFO mode, 1 = DMA */
    uint8_t polar_phase;  /* bit0=CPOL, bit1=CPHA (0b00=mode0, 0b01=mode1, 0b10=mode2, 0b11=mode3) */
    uint32_t freq;         /* SPI frequency in Hz */
    uint8_t pin_clk;       /* CLK pin number */
    uint8_t pin_mosi;      /* MOSI pin number */
    uint8_t pin_miso;      /* MISO pin number */
} hosal_spi_config_t;
```

### Example using HOSAL (Master Mode)

```c
#include "hosal_spi.h"

hosal_spi_dev_t spi0;

void spi0_example(void)
{
    /* Configure SPI0 as master, mode 0, 1MHz */
    spi0.port = 0;
    spi0.config.mode        = HOSAL_SPI_MODE_MASTER;
    spi0.config.dma_enable  = 0;
    spi0.config.polar_phase = 0;  /* CPOL=0, CPHA=0 (mode 0) */
    spi0.config.freq        = 1000000;
    spi0.config.pin_clk    = 12;
    spi0.config.pin_mosi   = 13;
    spi0.config.pin_miso   = 14;

    hosal_spi_init(&spi0);

    /* Send and receive simultaneously */
    uint8_t tx_buf[4] = {0x01, 0x02, 0x03, 0x04};
    uint8_t rx_buf[4];

    hosal_spi_send_recv(&spi0, tx_buf, rx_buf, 4, 1000);

    /* Send only */
    hosal_spi_send(&spi0, tx_buf, 4, 1000);

    /* Receive only */
    hosal_spi_recv(&spi0, rx_buf, 4, 1000);

    hosal_spi_finalize(&spi0);
}
```

### Example using HOSAL (Slave Mode)

```c
void spi_slave_example(void)
{
    hosal_spi_dev_t spi0;

    spi0.port = 0;
    spi0.config.mode        = HOSAL_SPI_MODE_SLAVE;
    spi0.config.dma_enable  = 0;
    spi0.config.polar_phase = 0;  /* must match master */
    spi0.config.freq        = 0;  /* ignored for slave */
    spi0.config.pin_clk    = 12;
    spi0.config.pin_mosi   = 13;
    spi0.config.pin_miso   = 14;

    hosal_spi_init(&spi0);

    uint8_t tx_buf[2] = {0xAA, 0x55};
    uint8_t rx_buf[2];

    /* Blocking exchange */
    hosal_spi_send_recv(&spi0, tx_buf, rx_buf, 2, HAL_WAIT_FOREVER);

    hosal_spi_finalize(&spi0);
}
```
