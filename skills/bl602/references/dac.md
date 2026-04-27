# BL602 DAC Register-Level API Reference

**Source File:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/gpip_reg.h`  
**HOSAL Header:** `components/platform/hosal/include/hosal_dac.h`  
**Base Address:** `GPIP_BASE = 0x40002000` (DAC is part of GPIP peripheral)  
**Note:** The BL602 has an 8-bit DAC for audio output. The DAC shares the GPIP peripheral with ADC.

---

## Register Overview

The DAC is part of the GPIP peripheral and provides 8-bit digital-to-analog conversion primarily for audio applications.

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x40` | `GPIP_DAC_CONFIG` | DAC configuration register |
| `0x44` | `GPIP_DAC_CONTROL` | DAC control and enable |
| `0x48` | `GPIP_DAC_DATA` | DAC data input |
| `0x4C` | `GPIP_DAC_FIFO_STATUS` | DAC FIFO status |
| `0x50` | `GPIP_DAC_IRQ_STATUS` | DAC interrupt status |
| `0x54` | `GPIP_DAC_IRQ_MASK` | DAC interrupt mask |
| `0x58` | `GPIP_DAC_TIMER_SCALE` | Sample rate timer scale |
| `0x5C` | `GPIP_DAC_TEST` | DAC test mode registers |

---

## Key Register Fields

### GPIP_DAC_CONFIG (Offset 0x40)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `DAC_EN` | [0] | DAC Enable | 1=enable DAC |
| `DAC_MODE` | [1] | Mode | 0=direct, 1=FIFO mode |
| `DAC_CH` | [2] | Channel | 0=left only, 1=right, 2=both |
| `DAC_REF_SEL` | [3] | Reference Select | 0=internal, 1=external |
| `DAC_SAMPLE_RATE` | [5:4] | Sample Rate | 0=32K, 1=44.1K, 2=48K, 3=96K |
| `DAC_INTR_EN` | [6] | Interrupt Enable | 1=enable interrupts |
| `DAC_FIFO_EN` | [7] | FIFO Enable | 1=enable FIFO mode |
| `DAC_FIFO_THRES` | [15:8] | FIFO Threshold | Interrupt threshold |
| `DAC_DMA_EN` | [16] | DMA Enable | 1=enable DMA |

### GPIP_DAC_CONTROL (Offset 0x44)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `DAC_START` | [0] | DAC Start | Write 1 to start conversion |
| `DAC_STOP` | [1] | DAC Stop | Write 1 to stop |
| `DAC_RST` | [2] | DAC Reset | Write 1 to reset |
| `DAC_CLR_FIFO` | [3] | Clear FIFO | Write 1 to clear FIFO |
| `DAC_RPT_EN` | [4] | Repeat Enable | 1=repeat data |

### GPIP_DAC_DATA (Offset 0x48)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `DAC_DATA` | [7:0] | DAC Data | 8-bit data to convert |
| `DAC_DATA_SIGN` | [8] | Sign Extension | Sign extend for audio |

### GPIP_DAC_FIFO_STATUS (Offset 0x4C)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `DAC_FIFO_COUNT` | [7:0] | FIFO Count | Number of samples in FIFO |
| `DAC_FIFO_FULL` | [8] | FIFO Full | 1=FIFO is full |
| `DAC_FIFO_EMPTY` | [9] | FIFO Empty | 1=FIFO is empty |
| `DAC_FIFO_OVER` | [10] | FIFO Overflow | 1=overflow occurred |
| `DAC_FIFO_UNDER` | [11] | FIFO Underflow | 1=underflow occurred |

### GPIP_DAC_TIMER_SCALE (Offset 0x58)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `DAC_TIMER_SCALE` | [15:0] | Timer Scale | Sample rate divisor |

---

## Register-Level Programming Example

```c
#include "bl602.h"
#include "gpip_reg.h"

#define GPIP_BASE  ((uint32_t)0x40002000)

/* Configure and start DAC in direct mode (no FIFO) */
void DAC_Start_Direct(uint8_t value)
{
    uint32_t tmpVal;

    /* Configure DAC: direct mode, 48K sample rate */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_DAC_CONFIG);
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_EN);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, DAC_MODE, 0);    /* direct mode */
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, DAC_SAMPLE_RATE, 2); /* 48K */
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_REF_SEL);         /* internal ref */
    BL_WR_REG(GPIP_BASE, GPIP_DAC_CONFIG, tmpVal);

    /* Set data and start */
    BL_WR_REG(GPIP_BASE, GPIP_DAC_DATA, value);

    /* Control start */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_DAC_CONTROL);
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_RST);
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_START);
    BL_WR_REG(GPIP_BASE, GPIP_DAC_CONTROL, tmpVal);
}

/* Configure DAC in FIFO mode for audio playback */
void DAC_Start_FIFO(void)
{
    uint32_t tmpVal;

    /* Configure DAC: FIFO mode, 48K, DMA enabled */
    tmpVal = 0;
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_MODE);            /* FIFO mode */
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_FIFO_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_DMA_EN);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, DAC_SAMPLE_RATE, 2); /* 48K */
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, DAC_FIFO_THRES, 16);
    BL_WR_REG(GPIP_BASE, GPIP_DAC_CONFIG, tmpVal);

    /* Reset and start */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_DAC_CONTROL);
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_RST);
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_CLR_FIFO);
    tmpVal = BL_SET_REG_BIT(tmpVal, DAC_START);
    BL_WR_REG(GPIP_BASE, GPIP_DAC_CONTROL, tmpVal);
}

/* Write a sample to DAC FIFO */
void DAC_FIFO_Write(uint8_t left_sample, uint8_t right_sample)
{
    /* In stereo mode, write left then right */
    BL_WR_REG(GPIP_BASE, GPIP_DAC_DATA, left_sample);
    BL_WR_REG(GPIP_BASE, GPIP_DAC_DATA, right_sample);
}

/* Generate a sine wave sample */
uint8_t DAC_Sine_Sample(uint32_t phase)
{
    /* 8-bit sine approximation: (128 + 127*sin(phase)) */
    int32_t sine = 127 * (int32_t)sin((double)phase * 2 * 3.14159 / 256.0);
    return (uint8_t)(128 + sine);
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_dac.h`

```c
/* HOSAL DAC device configuration */
typedef struct {
    uint8_t       dac_port;    /* DAC port number (0) */
    uint8_t       mode;        /* 0=direct, 1=FIFO, 2=DMA */
    uint32_t      sample_rate; /* sample rate in Hz */
    uint8_t       *buf;         /* data buffer (FIFO/DMA mode) */
    uint32_t      buf_len;     /* buffer length */
    void         *priv;        /* private data */
} hosal_dac_dev_t;

/* HOSAL DAC config */
typedef struct {
    uint8_t       mode;        /* 0=direct, 1=FIFO, 2=DMA */
    uint32_t      sample_rate; /* sample rate in Hz */
    uint8_t       channels;    /* 1=mono, 2=stereo */
    void         *priv;        /* private data */
} hosal_dac_config_t;

/**
 * @brief Initialize DAC device
 * @param dev  DAC device pointer
 * @return 0 on success, -1 on failure
 */
int hosal_dac_init(hosal_dac_dev_t *dev);

/**
 * @brief Start DAC output (direct mode)
 * @param dev  DAC device pointer
 * @param value  8-bit DAC value
 * @return 0 on success, -1 on failure
 */
int hosal_dac_start(hosal_dac_dev_t *dev, uint8_t value);

/**
 * @brief Start DAC output (FIFO/DMA mode)
 * @param dev  DAC device pointer
 * @return 0 on success, -1 on failure
 */
int hosal_dac_start_async(hosal_dac_dev_t *dev);

/**
 * @brief Write data to DAC
 * @param dev  DAC device pointer
 * @param buf  data buffer
 * @param len  number of bytes
 * @return number of bytes written, -1 on failure
 */
int hosal_dac_write(hosal_dac_dev_t *dev, uint8_t *buf, uint32_t len);

/**
 * @brief Get DAC status
 * @param dev  DAC device pointer
 * @param status  pointer to status variable
 * @return 0 on success, -1 on failure
 */
int hosal_dac_status_get(hosal_dac_dev_t *dev, uint32_t *status);

/**
 * @brief Deinitialize DAC device
 * @param dev  DAC device pointer
 * @return 0 on success, -1 on failure
 */
int hosal_dac_finalize(hosal_dac_dev_t *dev);
```

### HOSAL DAC Usage Example

```c
#include "hosal_dac.h"

hosal_dac_dev_t dac_dev;
uint8_t dac_buffer[256];

void dac_example(void)
{
    int ret;
    int i;

    /* Initialize DAC in direct mode */
    dac_dev.dac_port = 0;
    dac_dev.mode = 0;  /* direct mode */
    dac_dev.sample_rate = 48000;

    ret = hosal_dac_init(&dac_dev);
    if (ret != 0) {
        return;
    }

    /* Output a sawtooth wave */
    for (i = 0; i < 256; i++) {
        ret = hosal_dac_start(&dac_dev, i);
        /* Add delay for sample rate */
        for (volatile int j = 0; j < 1000; j++);
    }

    hosal_dac_finalize(&dac_dev);
}
```

### Notes

- The BL602 DAC is an 8-bit DAC primarily designed for audio output
- Supports sample rates from 32KHz to 96KHz
- Stereo mode requires writing left and right channel data alternately
- The DAC output can drive headphone or line-level loads with appropriate buffering
- For high-quality audio, use DMA mode to avoid CPU overhead
