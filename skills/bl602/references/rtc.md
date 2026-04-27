# BL602 RTC Register-Level API Reference

**Source File:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/hbn_reg.h`  
**HOSAL Header:** `components/platform/hosal/include/hosal_rtc.h`  
**Base Address:** `HBN_BASE = 0x4000F000` (Hibernate/AON module)  
**Note:** The RTC is part of the HBN (Hibernate) module, not a separate peripheral.

---

## Register Overview

The RTC is implemented within the HBN (Hibernate/AON) module and provides timekeeping that persists across power modes.

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x00` | `HBN_CTL` | HBN control (contains RTC enable) |
| `0x0C` | `HBN_RTC_TIME_L` | RTC time low 32 bits |
| `0x10` | `HBN_RTC_TIME_H` | RTC time high 8 bits + latch control |
| `0x14` | `HBN_IRQ_MODE` | HBN interrupt mode configuration |
| `0x18` | `HBN_IRQ_STAT` | HBN interrupt status |
| `0x1C` | `HBN_IRQ_CLR` | HBN interrupt clear |
| `0x20` | `HBN_PIR_CFG` | PIR configuration (if used for wakeup) |
| `0x100` | `HBN_TIME_L` | HBN timer low (power mode wakeup) |
| `0x104` | `HBN_TIME_H` | HBN timer high |

---

## Key Register Fields

### HBN_CTL (Offset 0x00)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `HBN_RTC_CTL` | [6:0] | RTC Control | RTC counter control bits |
| `HBN_MODE` | [7] | HBN Mode | 1=HBN mode selected |
| `HBN_PWRDN_HBN_RTC` | [11] | RTC Power Down | 0=RTC powered, 1=RTC off |
| `HBN_RTC_DLY_OPTION` | [24] | RTC Delay Option | Delay option for RTC start |
| `HBN_STATE` | [31:28] | HBN State | Current HBN state |

### HBN_RTC_TIME_L (Offset 0x0C)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `HBN_RTC_TIME_LATCH_L` | [31:0] | RTC Time Low | Lower 32 bits of RTC time (seconds) |

### HBN_RTC_TIME_H (Offset 0x10)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `HBN_RTC_TIME_LATCH_H` | [7:0] | RTC Time High | Upper 8 bits of RTC time |
| `HBN_RTC_TIME_LATCH` | [31] | RTC Latch Trigger | Write 1 to latch RTC time |

### HBN_IRQ_MODE (Offset 0x14)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `HBN_PIN_WAKEUP_MODE` | [2:0] | Pin Wakeup Mode | GPIO wakeup mode |
| `HBN_PIN_WAKEUP_MASK` | [4:3] | Pin Wakeup Mask | Pin wakeup mask |
| `HBN_IRQ_BOR_EN` | [18] | BOR Interrupt Enable | Brown-out reset interrupt enable |
| `HBN_PIN_WAKEUP_EN` | [27] | Pin Wakeup Enable | GPIO pin wakeup enable |

### HBN_IRQ_STAT (Offset 0x18)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `HBN_IRQ_STAT` | [31:0] | IRQ Status | Interrupt status bits |

---

## Register-Level Programming Example

```c
#include "bl602.h"
#include "hbn_reg.h"

#define HBN_BASE  ((uint32_t)0x4000F000)

/* Read current RTC time (64-bit seconds counter) */
uint64_t RTC_Get_Time(void)
{
    uint32_t time_h, time_l;

    /* Latch the RTC time */
    BL_WR_REG(HBN_BASE, HBN_RTC_TIME_H, (1 << 31));

    /* Read high then low (latched values) */
    time_h = BL_RD_REG(HBN_BASE, HBN_RTC_TIME_H);
    time_l = BL_RD_REG(HBN_BASE, HBN_RTC_TIME_L);

    /* Combine: time is in seconds, 40-bit counter */
    return ((uint64_t)(time_h & 0xFF) << 32) | time_l;
}

/* Set RTC time (40-bit seconds counter) */
void RTC_Set_Time(uint64_t seconds)
{
    uint32_t time_l = seconds & 0xFFFFFFFF;
    uint32_t time_h = (seconds >> 32) & 0xFF;

    /* Make sure RTC is powered */
    uint32_t tmpVal = BL_RD_REG(HBN_BASE, HBN_CTL);
    tmpVal = BL_CLR_REG_BIT(tmpVal, HBN_PWRDN_HBN_RTC);
    BL_WR_REG(HBN_BASE, HBN_CTL, tmpVal);

    /* Set time: write high first, then low triggers update */
    BL_WR_REG(HBN_BASE, HBN_RTC_TIME_H, time_h);
    BL_WR_REG(HBN_BASE, HBN_RTC_TIME_L, time_l);
}

/* Convert RTC seconds to calendar fields */
void RTC_Get_Calendar(uint64_t seconds, uint16_t *year, uint8_t *month,
                      uint8_t *day, uint8_t *hour, uint8_t *min, uint8_t *sec)
{
    uint32_t days, remaining;
    uint16_t y;
    static const uint8_t month_days[] = {31,28,31,30,31,30,31,31,30,31,30,31};

    days = seconds / 86400;  /* convert to days */
    remaining = seconds % 86400;

    *sec = remaining % 60;
    remaining /= 60;
    *min = remaining % 60;
    *hour = remaining / 60;

    /* Calculate year (simplified, not handling leap years perfectly) */
    for (y = 1970; days >= 365; y++) {
        if ((y % 4 == 0 && y % 100 != 0) || (y % 400 == 0)) {
            if (days >= 366) days -= 366;
            else break;
        } else {
            days -= 365;
        }
    }
    *year = y;

    /* Calculate month */
    uint8_t m;
    for (m = 1; m <= 12 && days >= month_days[m-1]; m++) {
        if (m == 2 && ((y % 4 == 0 && y % 100 != 0) || (y % 400 == 0))) {
            if (days >= 29) days -= 29;
            else break;
        } else {
            days -= month_days[m-1];
        }
    }
    *month = m;
    *day = days + 1;
}

/* Configure RTC alarm */
void RTC_Set_Alarm(uint64_t alarm_time, void (*callback)(void))
{
    uint32_t time_l = alarm_time & 0xFFFFFFFF;
    uint32_t time_h = (alarm_time >> 32) & 0xFF;

    /* Alarm would be configured through HBN interrupt mechanism */
    /* The alarm triggers an interrupt when RTC reaches alarm_time */

    /* Set alarm registers (platform-specific) */
    /* This is simplified - actual implementation may vary */
    BL_WR_REG(HBN_BASE, HBN_RTC_TIME_H, time_h);
    BL_WR_REG(HBN_BASE, HBN_RTC_TIME_L, time_l);
}

/* Initialize RTC */
void RTC_Init(void)
{
    uint32_t tmpVal;

    /* Enable RTC clock and power */
    tmpVal = BL_RD_REG(HBN_BASE, HBN_CTL);
    tmpVal = BL_CLR_REG_BIT(tmpVal, HBN_PWRDN_HBN_RTC);  /* power on RTC */
    /* Set RTC control - typically leave at default */
    BL_WR_REG(HBN_BASE, HBN_CTL, tmpVal);

    /* Wait for RTC to stabilize */
    for (volatile int i = 0; i < 1000; i++);
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_rtc.h`

```c
/* RTC time format options */
#define HOSAL_RTC_FORMAT_DEC 1  /* decimal format */
#define HOSAL_RTC_FORMAT_BCD 2  /* BCD format */

/* HOSAL RTC device configuration */
typedef struct {
    uint8_t       port;   /* rtc port (0) */
    hosal_rtc_config_t config; /* rtc config */
    void         *priv;   /* private data */
} hosal_rtc_dev_t;

/* HOSAL RTC configuration */
typedef struct {
    uint8_t format; /* time format DEC or BCD */
} hosal_rtc_config_t;

/* HOSAL RTC time structure */
typedef struct {
    uint8_t sec;     /* DEC: 0-59, BCD: 0x00-0x59 */
    uint8_t min;     /* DEC: 0-59, BCD: 0x00-0x59 */
    uint8_t hr;      /* DEC: 0-23, BCD: 0x00-0x23 */
    uint8_t date;    /* DEC: 1-31, BCD: 0x01-0x31 */
    uint8_t month;   /* DEC: 1-12, BCD: 0x01-0x12 */
    uint16_t year;   /* DEC: 0-9999, BCD: 0x0000-0x9999 */
} hosal_rtc_time_t;

/**
 * @brief Initialize the on-board CPU real time clock
 * @param rtc  rtc device pointer
 * @return 0 on success, other on failure
 */
int hosal_rtc_init(hosal_rtc_dev_t *rtc);

/**
 * @brief Set RTC time to a new value
 * @param rtc  rtc device pointer
 * @param time  pointer to time structure
 * @return 0 on success, other on failure
 */
int hosal_rtc_set_time(hosal_rtc_dev_t *rtc, const hosal_rtc_time_t *time);

/**
 * @brief Get current RTC time
 * @param rtc  rtc device pointer
 * @param time  pointer to time structure
 * @return 0 on success, other on failure
 */
int hosal_rtc_get_time(hosal_rtc_dev_t *rtc, hosal_rtc_time_t *time);

/**
 * @brief Set RTC time as 64-bit timestamp
 * @param rtc  rtc device pointer
 * @param time_stamp  new time value (seconds since epoch)
 * @return 0 on success, other on failure
 */
int hosal_rtc_set_count(hosal_rtc_dev_t *rtc, uint64_t *time_stamp);

/**
 * @brief Get RTC time as 64-bit timestamp
 * @param rtc  rtc device pointer
 * @param time_stamp  pointer to store timestamp
 * @return 0 on success, other on failure
 */
int hosal_rtc_get_count(hosal_rtc_dev_t *rtc, uint64_t *time_stamp);

/**
 * @brief Deinitialize RTC interface
 * @param rtc  rtc device pointer
 * @return 0 on success, other on failure
 */
int hosal_rtc_finalize(hosal_rtc_dev_t *rtc);
```

### HOSAL RTC Usage Example

```c
#include "hosal_rtc.h"

hosal_rtc_dev_t rtc_dev;

void rtc_example(void)
{
    int ret;
    hosal_rtc_time_t time;

    /* Initialize RTC in DEC mode */
    rtc_dev.port = 0;
    rtc_dev.config.format = HOSAL_RTC_FORMAT_DEC;

    ret = hosal_rtc_init(&rtc_dev);
    if (ret != 0) {
        printf("RTC init failed\r\n");
        return;
    }

    /* Set time: 2024-01-15 10:30:00 */
    time.year = 2024;
    time.month = 1;
    time.date = 15;
    time.hr = 10;
    time.min = 30;
    time.sec = 0;

    ret = hosal_rtc_set_time(&rtc_dev, &time);
    if (ret == 0) {
        printf("Time set successfully\r\n");
    }

    /* Get and display current time */
    ret = hosal_rtc_get_time(&rtc_dev, &time);
    if (ret == 0) {
        printf("%04u-%02u-%02u %02u:%02u:%02u\r\n",
               time.year, time.month, time.date,
               time.hr, time.min, time.sec);
    }

    /* Using timestamp interface */
    uint64_t timestamp;
    hosal_rtc_get_count(&rtc_dev, &timestamp);
    printf("RTC timestamp: %llu seconds\r\n", timestamp);

    hosal_rtc_finalize(&rtc_dev);
}
```

### Notes

- The RTC is part of the HBN (Hibernate) always-on domain and persists across power modes
- The RTC counter is 40 bits wide, allowing for very long time periods (hundreds of years)
- Time is stored as seconds since an epoch (typically 2000-01-01 or 1970-01-01)
- The RTC can continue running in hibernate mode with very low power consumption
- Alarm functionality allows wakeup from hibernate at a specified time
- The RTC clock source is typically 32.768KHz from an external crystal
- Calibration may be needed to achieve accurate timekeeping
- RTC time is stored in binary format internally (seconds counter), converted to calendar for user
