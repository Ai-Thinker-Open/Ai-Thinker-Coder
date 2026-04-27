# BL602 RNG (TRNG) Register-Level API Reference

**Source File:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/sec_eng_reg.h`  
**HOSAL Header:** `components/platform/hosal/include/hosal_rng.h`  
**Base Address:** `SEC_ENG_BASE = 0x40004000` (Security Engine module)  
**TRNG Offset:** `0x200` (TRNG block within SEC_ENG)  
**Interrupt:** `SEC_TRNG_IRQn` (IRQ 28)  
**Driver Source:** `components/platform/soc/bl602/bl602_std/bl602_std/StdDriver/Src/bl602_sec_eng.c`

---

## Register Overview

The RNG is implemented as a True Random Number Generator (TRNG) within the Security Engine. It uses analog noise sources and provides cryptographically secure random numbers.

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x200` | `SEC_ENG_SE_TRNG_0_CTRL_0` | TRNG control and status |
| `0x204` | `SEC_ENG_SE_TRNG_0_STATUS` | TRNG status register |
| `0x208` | `SEC_ENG_SE_TRNG_0_DOUT_0` | Random data output [31:0] |
| `0x20C` | `SEC_ENG_SE_TRNG_0_DOUT_1` | Random data output [63:32] |
| `0x210` | `SEC_ENG_SE_TRNG_0_DOUT_2` | Random data output [95:64] |
| `0x214` | `SEC_ENG_SE_TRNG_0_DOUT_3` | Random data output [127:96] |
| `0x218` | `SEC_ENG_SE_TRNG_0_DOUT_4` | Random data output [159:128] |
| `0x21C` | `SEC_ENG_SE_TRNG_0_DOUT_5` | Random data output [191:160] |
| `0x220` | `SEC_ENG_SE_TRNG_0_DOUT_6` | Random data output [223:192] |
| `0x224` | `SEC_ENG_SE_TRNG_0_DOUT_7` | Random data output [255:224] |

---

## Key Register Fields

### SEC_ENG_SE_TRNG_0_CTRL_0 (Offset 0x200)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `SE_TRNG_0_BUSY` | [0] | TRNG Busy | 1=TRNG is generating numbers |
| `SE_TRNG_0_TRIG_1T` | [1] | Trigger | Write 1 to trigger generation |
| `SE_TRNG_0_EN` | [2] | TRNG Enable | 1=enable TRNG |
| `SE_TRNG_0_DOUT_CLR_1T` | [3] | Clear Output | Write 1 to clear output |
| `SE_TRNG_0_HT_ERROR` | [4] | Health Test Error | 1=health test failed |
| `SE_TRNG_0_INT` | [8] | Interrupt Flag | 1=TRNG interrupt pending |
| `SE_TRNG_0_INT_CLR_1T` | [9] | Clear Interrupt | Write 1 to clear interrupt |
| `SE_TRNG_0_INT_SET_1T` | [10] | Set Interrupt | Write 1 to set interrupt |
| `SE_TRNG_0_INT_MASK` | [11] | Interrupt Mask | 1=mask interrupt |
| `SE_TRNG_0_MANUAL_FUN_SEL` | [13] | Manual Mode | 1=use manual function |
| `SE_TRNG_0_MANUAL_RESEED` | [14] | Manual Reseed | Write 1 to reseed |
| `SE_TRNG_0_MANUAL_EN` | [15] | Manual Enable | 1=enable manual mode |

### SEC_ENG_SE_TRNG_0_STATUS (Offset 0x204)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `SE_TRNG_0_STATUS` | [31:0] | TRNG Status | Status bits from TRNG health check |

---

## Register-Level Programming Example

```c
#include "bl602.h"
#include "sec_eng_reg.h"

#define SEC_ENG_BASE  ((uint32_t)0x40004000)

/* Check if TRNG is busy */
uint8_t TRNG_Is_Busy(void)
{
    uint32_t val = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0);
    return BL_IS_REG_BIT_SET(val, SE_TRNG_0_BUSY);
}

/* Generate random 32-bit number (blocking) */
uint32_t TRNG_Read_32(void)
{
    uint32_t tmpVal;

    /* Enable and trigger TRNG */
    tmpVal = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0);
    tmpVal = BL_SET_REG_BIT(tmpVal, SE_TRNG_0_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, SE_TRNG_0_TRIG_1T);
    BL_WR_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0, tmpVal);

    /* Wait for completion */
    do {
        tmpVal = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0);
    } while (BL_IS_REG_BIT_SET(tmpVal, SE_TRNG_0_BUSY));

    /* Read random data */
    return BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_DOUT_0);
}

/* Generate random 64-bit number (blocking) */
uint64_t TRNG_Read_64(void)
{
    uint32_t tmpVal;
    uint32_t lo, hi;

    /* Enable and trigger */
    tmpVal = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0);
    tmpVal = BL_SET_REG_BIT(tmpVal, SE_TRNG_0_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, SE_TRNG_0_TRIG_1T);
    BL_WR_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0, tmpVal);

    /* Wait for completion */
    do {
        tmpVal = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0);
    } while (BL_IS_REG_BIT_SET(tmpVal, SE_TRNG_0_BUSY));

    /* Read 64 bits */
    lo = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_DOUT_0);
    hi = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_DOUT_1);

    return ((uint64_t)hi << 32) | lo;
}

/* Generate 256-bit random number (blocking) */
void TRNG_Read_256(uint8_t *buf)
{
    uint32_t tmpVal;
    uint32_t *words = (uint32_t *)buf;
    int i;

    /* Enable and trigger */
    tmpVal = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0);
    tmpVal = BL_SET_REG_BIT(tmpVal, SE_TRNG_0_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, SE_TRNG_0_TRIG_1T);
    BL_WR_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0, tmpVal);

    /* Wait for completion */
    do {
        tmpVal = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0);
    } while (BL_IS_REG_BIT_SET(tmpVal, SE_TRNG_0_BUSY));

    /* Read all 8 words */
    for (i = 0; i < 8; i++) {
        words[i] = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_DOUT_0 + i * 4);
    }
}

/* Fill buffer with random data */
void TRNG_Fill_Buffer(uint8_t *buf, uint32_t len)
{
    uint32_t words_len = len / 4;
    uint32_t remaining = len % 4;
    uint32_t i;

    /* Fill complete 32-bit words */
    for (i = 0; i < words_len; i++) {
        ((uint32_t *)buf)[i] = TRNG_Read_32();
    }

    /* Handle remaining bytes */
    if (remaining > 0) {
        uint32_t last_word = TRNG_Read_32();
        for (i = 0; i < remaining; i++) {
            buf[words_len * 4 + i] = (last_word >> (i * 8)) & 0xFF;
        }
    }
}

/* Check TRNG health test status */
uint8_t TRNG_Health_Test_Pass(void)
{
    uint32_t ctrl = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_CTRL_0);
    uint32_t status = BL_RD_REG(SEC_ENG_BASE, SEC_ENG_SE_TRNG_0_STATUS);

    /* HT_ERROR bit should be 0 if health test passed */
    if (BL_IS_REG_BIT_SET(ctrl, SE_TRNG_0_HT_ERROR)) {
        return 0;  /* failed */
    }

    /* Check specific health test bits in status register */
    /* (bits vary by implementation, check datasheet) */
    return (status != 0xFFFFFFFF) ? 1 : 0;
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_rng.h`

```c
/**
 * @brief Initialize RNG
 * @return 0 on success, other on failure
 */
int hosal_rng_init(void);

/**
 * @brief Fill a memory buffer with random data
 * @param buf  pointer to memory buffer
 * @param bytes  number of bytes to read
 * @return 0 on success, other on failure
 */
int hosal_random_num_read(void *buf, uint32_t bytes);
```

### HOSAL RNG Usage Example

```c
#include "hosal_rng.h"
#include <string.h>

void rng_example(void)
{
    int ret;
    uint32_t random_word;
    uint8_t random_bytes[32];
    uint8_t key[16];

    /* Initialize RNG */
    ret = hosal_rng_init();
    if (ret != 0) {
        printf("RNG init failed\r\n");
        return;
    }

    /* Read single 32-bit random number */
    ret = hosal_random_num_read(&random_word, sizeof(random_word));
    if (ret == 0) {
        printf("Random: 0x%08X\r\n", random_word);
    }

    /* Fill buffer with random bytes */
    ret = hosal_random_num_read(random_bytes, sizeof(random_bytes));
    if (ret == 0) {
        printf("First 8 bytes: %02X %02X %02X %02X %02X %02X %02X %02X\r\n",
               random_bytes[0], random_bytes[1], random_bytes[2], random_bytes[3],
               random_bytes[4], random_bytes[5], random_bytes[6], random_bytes[7]);
    }

    /* Generate AES key */
    ret = hosal_random_num_read(key, sizeof(key));
    if (ret == 0) {
        printf("AES-128 key generated\r\n");
        /* key is ready for use with sec_eng AES */
    }

    /* Generate random MAC address */
    uint8_t mac[6];
    ret = hosal_random_num_read(mac, 6);
    if (ret == 0) {
        /* Ensure multicast bit is clear and local bit is set */
        mac[0] = (mac[0] & 0xFE) | 0x02;
        printf("Random MAC: %02X:%02X:%02X:%02X:%02X:%02X\r\n",
               mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
    }

    /* Generate random password/salt */
    uint8_t salt[16];
    hosal_random_num_read(salt, sizeof(salt));
    printf("Random salt generated for password hashing\r\n");
}
```

### Notes

- The TRNG uses analog noise sources (typically thermal noise from resistors) for true randomness
- The TRNG includes built-in health tests to ensure statistical quality of output
- The TRNG output is suitable for cryptographic operations including key generation
- For high-throughput applications, use the HOSAL `hosal_random_num_read()` which handles blocking and buffering
- The TRNG can generate up to 256 bits (8 words) per request
- Health test errors (HT_ERROR bit) indicate the analog source may be malfunctioning
- The RNG is part of the always-on security engine and remains available in low-power modes
- For very high-speed random number generation, consider using DMA mode if supported
