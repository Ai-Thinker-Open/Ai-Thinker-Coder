# BL602 ADC Register-Level API Reference

**Source File:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/gpip_reg.h`  
**HOSAL Header:** `components/platform/hosal/include/hosal_adc.h`  
**Base Address:** `GPIP_BASE = 0x40002000` (AUX/GPIO combined module)  
**Driver Source:** `components/platform/soc/bl602/bl602_std/bl602_std/StdDriver/Src/bl602_adc.c`

---

## Register Overview

The ADC is part of the GPIP (GPIO/AUX) peripheral. It supports 12-bit SAR ADC with multiple channels, DMA support, and interrupt capability.

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x00` | `GPIP_GPADC_CONFIG` | ADC configuration and control |
| `0x04` | `GPIP_GPADC_GLOBAL` | ADC global enable and clock |
| `0x08` | `GPIP_GPADC_STATUS` | ADC status flags |
| `0x0C` | `GPIP_GPADC_RDATA` | ADC raw data output |
| `0x10` | `GPIP_GPADC_DMA_RDATA` | ADC DMA read data |
| `0x14` | `GPIP_GPADC_IRQ_STATUS` | ADC interrupt status |
| `0x18` | `GPIP_GPADC_FIFO` | ADC FIFO data (if available) |
| `0x1C` | `GPIP_GPADC_FIFO_IRQ_CFG` | FIFO interrupt configuration |
| `0x20` | `GPIP_GPADC_CONTINUOUS_CONFIG` | Continuous conversion config |
| `0x24` | `GPIP_GPADC_CH_LIST` | Channel selection list |
| `0x28` | `GPIP_GPADC_CH_DISABLE` | Channel disable mask |
| `0x2C` | `GPIP_GPADC_CH_CONFIG` | Per-channel configuration |
| `0x30` | `GPIP_GPADC_SW_CONFIG` | Software trigger configuration |
| `0x34` | `GPIP_GPADC_THREHOLD` | High/low threshold for DMA/interrupt |
| `0x38` | `GPIP_GPADC_CALIBRATION` | Calibration value |

---

## Key Register Fields

### GPIP_GPADC_CONFIG (Offset 0x00)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `GPADC_EN` | [0] | ADC Enable | 1=enable ADC |
| `GPADC_CH` | [7:4] | Channel Select | 0-7 for channels 0-7 |
| `GPADC_MODE` | [9:8] | Conversion Mode | 0=single, 1=continuous |
| `GPADC_CLK` | [11:10] | Clock Divider | 0=/1, 1=/2, 2=/4, 3=/8 |
| `GPADC_DMA_EN` | [12] | DMA Enable | 1=enable DMA |
| `GPADC_IRQ_EN` | [13] | Interrupt Enable | 1=enable interrupt |
| `GPADC_START` | [16] | Start Conversion | Write 1 to trigger |
| `GPADC_CONTINUOUS` | [17] | Continuous Mode | 1=continuous conversion |
| `GPADC_RJCT` | [19:18] | Clock Rejection | Noise rejection setting |
| `GPADC_PGA_EN` | [20] | PGA Enable | 1=enable programmable gain amp |
| `GPADC_PGA_VSEL` | [23:21] | PGA Voltage Select | Gain selection |
| `GPADC_BKUP_DATA` | [31:24] | Backup Data | Calibration backup |

### GPIP_GPADC_GLOBAL (Offset 0x04)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `GPADC_CORE_EN` | [0] | Core Enable | Global ADC core power enable |
| `GPADC_SAMPLE_DLY` | [7:4] | Sample Delay | Delay between samples (cycles) |
| `GPADC_STORE_EN` | [8] | Store Enable | Store results to FIFO |

### GPIP_GPADC_STATUS (Offset 0x08)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `GPADC_BUSY` | [0] | ADC Busy | 1=ADC is converting |
| `GPADC_DATA_VALID` | [1] | Data Valid | 1=result ready |
| `GPADC_FIFO_FULL` | [2] | FIFO Full | 1=FIFO is full |
| `GPADC_FIFO_EMPTY` | [3] | FIFO Empty | 1=FIFO is empty |
| `GPADC_CALI_DONE` | [4] | Calibration Done | 1=calibration complete |

### GPIP_GPADC_RDATA (Offset 0x0C)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `GPADC_RDATA` | [11:0] | ADC Result | 12-bit ADC result |
| `GPADC_CH` | [15:12] | Channel | Source channel of result |
| `GPADC_DONE` | [16] | Conversion Done | One-shot done flag |

### GPIP_GPADC_CH_LIST (Offset 0x24)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `GPADC_CH_LIST` | [31:0] | Channel List | Bitmask of enabled channels (bit N = channel N) |

---

## Register-Level Programming Example

```c
#include "bl602.h"
#include "gpip_reg.h"

#define GPIP_BASE  ((uint32_t)0x40002000)

/* Read ADC single channel, channel 0 */
uint16_t ADC_Read_Single(uint8_t channel)
{
    uint32_t tmpVal;

    /* Configure ADC: single conversion, enable */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_GPADC_CONFIG);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, GPADC_EN, 1);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, GPADC_CH, channel);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, GPADC_MODE, 0);  /* single mode */
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, GPADC_CONTINUOUS, 0);
    BL_WR_REG(GPIP_BASE, GPIP_GPADC_CONFIG, tmpVal);

    /* Enable global core */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_GPADC_GLOBAL);
    tmpVal = BL_SET_REG_BIT(tmpVal, GPADC_CORE_EN);
    BL_WR_REG(GPIP_BASE, GPIP_GPADC_GLOBAL, tmpVal);

    /* Trigger conversion */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_GPADC_CONFIG);
    tmpVal = BL_SET_REG_BIT(tmpVal, GPADC_START);
    BL_WR_REG(GPIP_BASE, GPIP_GPADC_CONFIG, tmpVal);

    /* Wait for completion */
    do {
        tmpVal = BL_RD_REG(GPIP_BASE, GPIP_GPADC_STATUS);
    } while (!BL_IS_REG_BIT_SET(tmpVal, GPADC_DATA_VALID));

    /* Read result */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_GPADC_RDATA);
    return (uint16_t)(tmpVal & 0xFFF);
}

/* Read all 8 channels in sequence */
void ADC_Read_All_Channels(uint16_t results[8])
{
    uint32_t tmpVal;
    int i;

    /* Enable all 8 channels in list */
    BL_WR_REG(GPIP_BASE, GPIP_GPADC_CH_LIST, 0xFF);

    /* Configure continuous mode */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_GPADC_CONFIG);
    tmpVal = BL_SET_REG_BIT(tmpVal, GPADC_CORE_EN);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, GPADC_MODE, 1);  /* continuous */
    tmpVal = BL_SET_REG_BIT(tmpVal, GPADC_CONTINUOUS);
    BL_WR_REG(GPIP_BASE, GPIP_GPADC_CONFIG, tmpVal);

    /* Start conversion */
    tmpVal = BL_RD_REG(GPIP_BASE, GPIP_GPADC_CONFIG);
    tmpVal = BL_SET_REG_BIT(tmpVal, GPADC_START);
    BL_WR_REG(GPIP_BASE, GPIP_GPADC_CONFIG, tmpVal);

    /* Read all channels */
    for (i = 0; i < 8; i++) {
        do {
            tmpVal = BL_RD_REG(GPIP_BASE, GPIP_GPADC_STATUS);
        } while (!BL_IS_REG_BIT_SET(tmpVal, GPADC_DATA_VALID));
        results[i] = (uint16_t)(BL_RD_REG(GPIP_BASE, GPIP_GPADC_RDATA) & 0xFFF);
    }
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_adc.h`

```c
/* HOSAL ADC device configuration */
typedef struct {
    uint8_t       adc_port;    /* ADC port number */
    uint8_t       channel;     /* ADC channel 0-7 */
    uint16_t     *buf;         /* data buffer (DMA mode) */
    uint16_t      buf_len;     /* buffer length */
    void         *priv;        /* private data */
} hosal_adc_dev_t;

/* HOSAL ADC attributes */
typedef struct {
    uint8_t       channel;     /* current channel */
    uint8_t       continuous; /* 0=single shot, 1=continuous */
    uint16_t     *buf;         /* current buffer */
    uint16_t      buf_len;     /* buffer length */
    void         *priv;        /* private data */
} hosal_adc_attribute_t;

/**
 * @brief Initialize ADC device
 * @param dev  ADC device pointer
 * @return 0 on success, -1 on failure
 */
int hosal_adc_init(hosal_adc_dev_t *dev);

/**
 * @brief Start ADC conversion (polling mode)
 * @param dev  ADC device pointer
 * @param value  pointer to store result
 * @param timeout  timeout in ms (0 for infinite)
 * @return 0 on success, -1 on failure
 */
int hosal_adc_start(hosal_adc_dev_t *dev, uint16_t *value, uint32_t timeout);

/**
 * @brief Start ADC with DMA (async mode)
 * @param dev  ADC device pointer
 * @return 0 on success, -1 on failure
 */
int hosal_adc_start_async(hosal_adc_dev_t *dev);

/**
 * @brief Get ADC conversion result
 * @param dev  ADC device pointer
 * @param value  pointer to store result
 * @return 0 on success, -1 on failure
 */
int hosal_adc_value_get(hosal_adc_dev_t *dev, uint16_t *value);

/**
 * @brief Get ADC full attribute
 * @param dev  ADC device pointer
 * @param attr  attribute structure pointer
 * @return 0 on success, -1 on failure
 */
int hosal_adc_attr_get(hosal_adc_dev_t *dev, hosal_adc_attribute_t *attr);

/**
 * @brief Set ADC attribute
 * @param dev  ADC device pointer
 * @param attr  attribute structure pointer
 * @return 0 on success, -1 on failure
 */
int hosal_adc_attr_set(hosal_adc_dev_t *dev, hosal_adc_attribute_t *attr);

/**
 * @brief Deinitialize ADC device
 * @param dev  ADC device pointer
 * @return 0 on success, -1 on failure
 */
int hosal_adc_finalize(hosal_adc_dev_t *dev);
```

### HOSAL ADC Usage Example

```c
#include "hosal_adc.h"

hosal_adc_dev_t adc_dev;

void adc_example(void)
{
    uint16_t value;
    int ret;

    /* Initialize ADC on channel 0 */
    adc_dev.adc_port = 0;
    adc_dev.channel = 0;

    ret = hosal_adc_init(&adc_dev);
    if (ret != 0) {
        /* handle error */
        return;
    }

    /* Single conversion */
    ret = hosal_adc_start(&adc_dev, &value, 1000);
    if (ret == 0) {
        printf("ADC channel 0 value: %u\r\n", value);
    }

    hosal_adc_finalize(&adc_dev);
}
```

### Notes

- The ADC uses a 12-bit SAR architecture with input range 0 to VDD (typically 3.3V)
- Supports up to 8 external channels plus internal temperature sensor
- Calibration is performed at factory and values are stored in eFuse
- Maximum sample rate depends on clock divider; typical at 32MHz PCLK with /8 divider gives ~250KSPS
