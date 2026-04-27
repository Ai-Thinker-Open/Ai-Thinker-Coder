# RTC API 参考

> 来源文件：`components/platform/hosal/include/hosal_rtc.h`

## 宏定义

```c
#define HOSAL_RTC_FORMAT_DEC 1  // 十进制格式
#define HOSAL_RTC_FORMAT_BCD 2  // BCD 格式
```

## 类型定义

### `hosal_rtc_config_t` — RTC 配置结构

```c
typedef struct {
    uint8_t format;  // 时间格式：HOSAL_RTC_FORMAT_DEC 或 HOSAL_RTC_FORMAT_BCD
} hosal_rtc_config_t;
```

### `hosal_rtc_time_t` — RTC 时间结构

```c
typedef struct {
    uint8_t  sec;     // 秒（DEC: 0~59 / BCD: 0x00~0x59）
    uint8_t  min;     // 分（DEC: 0~59 / BCD: 0x00~0x59）
    uint8_t  hr;      // 时（DEC: 0~23 / BCD: 0x00~0x23）
    uint8_t  date;    // 日（DEC: 1~31 / BCD: 0x01~0x31）
    uint8_t  month;   // 月（DEC: 1~12 / BCD: 0x01~0x12）
    uint16_t year;    // 年（DEC: 0~9999 / BCD: 0x0000~0x9999）
} hosal_rtc_time_t;
```

### `hosal_rtc_dev_t` — RTC 设备结构

```c
typedef struct {
    uint8_t       port;
    hosal_rtc_config_t  config;
    void         *priv;
} hosal_rtc_dev_t;
```

## 函数接口

### `hosal_rtc_init`

初始化 RTC。

```c
int hosal_rtc_init(hosal_rtc_dev_t *rtc);
```

---

### `hosal_rtc_set_time`

设置时间（struct 方式）。

```c
int hosal_rtc_set_time(hosal_rtc_dev_t *rtc, const hosal_rtc_time_t *time);
```

---

### `hosal_rtc_get_time`

读取时间（struct 方式）。

```c
int hosal_rtc_get_time(hosal_rtc_dev_t *rtc, hosal_rtc_time_t *time);
```

---

### `hosal_rtc_set_count`

设置时间（时间戳方式）。

```c
int hosal_rtc_set_count(hosal_rtc_dev_t *rtc, uint64_t *time_stamp);
```

| 参数 | 说明 |
|------|------|
| `time_stamp` | 64 位时间戳（秒） |

---

### `hosal_rtc_get_count`

读取时间（时间戳方式）。

```c
int hosal_rtc_get_count(hosal_rtc_dev_t *rtc, uint64_t *time_stamp);
```

---

### `hosal_rtc_finalize`

释放 RTC。

```c
int hosal_rtc_finalize(hosal_rtc_dev_t *rtc);
```

## 使用示例

```c
#include "hal_rtc.h"

hosal_rtc_dev_t rtc0 = {
    .port = 0,
    .config = {
        .format = HOSAL_RTC_FORMAT_DEC,  // 十进制格式
    }
};

hosal_rtc_init(&rtc0);

// 设置时间
hosal_rtc_time_t set_time = {
    .year = 2025,
    .month = 6,
    .date = 18,
    .hr = 10,
    .min = 30,
    .sec = 0,
};
hosal_rtc_set_time(&rtc0, &set_time);

// 读取时间
hosal_rtc_time_t cur_time;
hosal_rtc_get_time(&rtc0, &cur_time);
printf("%04d-%02d-%02d %02d:%02d:%02d\r\n",
       cur_time.year, cur_time.month, cur_time.date,
       cur_time.hr, cur_time.min, cur_time.sec);

// 时间戳方式
uint64_t ts;
hosal_rtc_get_count(&rtc0, &ts);
printf("Timestamp: %llu\r\n", ts);

// 关闭
hosal_rtc_finalize(&rtc0);
```
