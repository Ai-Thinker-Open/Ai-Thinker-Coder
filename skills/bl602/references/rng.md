# RNG API Reference

> Source file: `components/platform/hosal/include/hosal_rng.h`

## Function Interface

### `hosal_rng_init`

Initialize the random number generator.

```c
int hosal_rng_init(void);
```

**Return value**: `0` success, others failure

> This function must be called before `hosal_random_num_read`.

---

### `hosal_random_num_read`

Read random numbers to fill the buffer.

```c
int hosal_random_num_read(void *buf, uint32_t bytes);
```

| Parameter | Description |
|-----------|-------------|
| `buf` | Valid memory buffer, random numbers will be filled into this memory |
| `bytes` | Buffer length (bytes) |

**Return value**: `0` success, others failure

## Usage Example

```c
#include "hal_rng.h"

// Initialize RNG (usually called once during system initialization)
hosal_rng_init();

// Read 8 bytes of random numbers
uint8_t random_bytes[8];
int ret = hosal_random_num_read(random_bytes, 8);
if (ret == 0) {
    printf("Random: %02X%02X%02X%02X%02X%02X%02X%02X\r\n",
           random_bytes[0], random_bytes[1], random_bytes[2], random_bytes[3],
           random_bytes[4], random_bytes[5], random_bytes[6], random_bytes[7]);
}

// Generate random numbers for keys, random delays, frequency hopping, etc.
uint32_t random_val;
hosal_random_num_read(&random_val, sizeof(random_val));
```

## Application Scenarios

| Scenario | Description |
|----------|-------------|
| Key generation | Session keys for encrypted communication |
| Random delay | Random backoff time to avoid wireless communication conflicts |
| MAC address | Generate random MAC address for testing |
| Frequency hopping seed | Pseudo-random sequence seed for frequency hopping communication |
