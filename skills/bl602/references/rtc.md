# RTC API Reference

> Source file: `components/platform/hosal/include/hosal_rtc.h`

## Macros

```c
#define HOSAL_RTC_FORMAT_DEC 1  // Decimal format
#define HOSAL_RTC_FORMAT_BCD 2  // BCD format
```

## Type Definitions

### `hosal_rtc_config_t` — RTC Configuration Structure

```c
typedef struct {
    uint8_t format;  // Time format: HOSAL_RTC_FORMAT_DEC or HOSAL_RTC_FORMAT_BCD
} hosal_rtc_config_t;
```

### `hosal_rtc_time_t` — RTC Time Structure

```c
typedef struct {
    uint8_t  sec;     // Seconds (DEC: 0~59 / BCD: 0x00~0x59)
    uint8_t  min;     // Minutes (DEC: 0~59 / BCD: 0x00~0x59)
    uint8_t  hr;      // Hours (DEC: 0~23 / BCD: 0x00~0x23)
    uint8_t  date;    // Day (DEC: 1~31 / BCD: 0x01~0x31)
    uint8_t  month;   // Month (DEC: 1~12 / BCD: 0x01~0x12)
    uint16_t year;    // Year (DEC: 0~9999 / BCD: 0x0000~0x9999)
} hosal_rtc_time_t;
```

### `hosal_rtc_dev_t` — RTC Device Structure

```c
typedef struct {
    uint8_t       port;
    hosal_rtc_config_t  config;
    void         *priv;
} hosal_rtc_dev_t;
```

## Function API

### `hosal_rtc_init`

Initialize RTC.

```c
int hosal_rtc_init(hosal_rtc_dev_t *rtc);
```

---

### `hosal_rtc_set_time`

Set time (struct mode).

```c
int hosal_rtc_set_time(hosal_rtc_dev_t *rtc, const hosal_rtc_time_t *time);
```

---

### `hosal_rtc_get_time`

Read time (struct mode).

```c
int hosal_rtc_get_time(hosal_rtc_dev_t *rtc, hosal_rtc_time_t *time);
```

---

### `hosal_rtc_set_count`

Set time (timestamp mode).

```c
int hosal_rtc_set_count(hosal_rtc_dev_t *rtc, uint64_t *time_stamp);
```

| Parameter | Description |
|-----------|-------------|
| `time_stamp` | 64-bit timestamp (seconds) |

---

### `hosal_rtc_get_count`

Read time (timestamp mode).

```c
int hosal_rtc_get_count(hosal_rtc_dev_t *rtc, uint64_t *time_stamp);
```

---

### `hosal_rtc_finalize`

Finalize RTC.

```c
int hosal_rtc_finalize(hosal_rtc_dev_t *rtc);
```

## Usage Example

```c
#include "hal_rtc.h"

hosal_rtc_dev_t rtc0 = {
    .port = 0,
    .config = {
        .format = HOSAL_RTC_FORMAT_DEC,  // Decimal format
    }
};

hosal_rtc_init(&rtc0);

// Set time
hosal_rtc_time_t set_time = {
    .year = 2025,
    .month = 6,
    .date = 18,
    .hr = 10,
    .min = 30,
    .sec = 0,
};
hosal_rtc_set_time(&rtc0, &set_time);

// Read time
hosal_rtc_time_t cur_time;
hosal_rtc_get_time(&rtc0, &cur_time);
printf("%04d-%02d-%02d %02d:%02d:%02d\r\n",
       cur_time.year, cur_time.month, cur_time.date,
       cur_time.hr, cur_time.min, cur_time.sec);

// Timestamp mode
uint64_t ts;
hosal_rtc_get_count(&rtc0, &ts);
printf("Timestamp: %llu\r\n", ts);

// Finalize
hosal_rtc_finalize(&rtc0);
```
