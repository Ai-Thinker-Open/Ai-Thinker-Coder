# BL602 Flash Register-Level API Reference

**Source File:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/sf_ctrl_reg.h`  
**HOSAL Header:** `components/platform/hosal/include/hosal_flash.h`  
**Base Address:** `SF_CTRL_BASE = 0x4000B000`  
**Driver Source:** `components/platform/soc/bl602/bl602_std/bl602_std/StdDriver/Src/bl602_sf_ctrl.c`

---

## Register Overview

The Serial Flash Controller (SF_CTRL) manages external SPI flash memory. It supports XIP (Execute-In-Place), direct read/write, and flash encryption.

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x00` | `SF_CTRL_CFG` | Flash controller configuration |
| `0x04` | `SF_CTRL_STATUS` | Flash controller status |
| `0x08` | `SF_CTRL_CMD` | Flash command register |
| `0x0C` | `SF_CTRL_ADDR` | Flash address register |
| `0x10` | `SF_CTRL_TXDATA` | TX data register (32 bytes max) |
| `0x14` | `SF_CTRL_RXDATA` | RX data register (32 bytes max) |
| `0x18` | `SF_CTRL_CTRL` | Control register (start, busy, etc.) |
| `0x1C` | `SF_CTRL_INT_STATUS` | Interrupt status |
| `0x20` | `SF_CTRL_INT_MASK` | Interrupt mask |
| `0x24` | `SF_CTRL_TIMING` | Flash timing configuration |
| `0x28` | `SF_CTRL_SF_ID` | Flash JEDEC ID |
| `0x2C` | `SF_CTRL_FIFO_CTRL` | FIFO control |
| `0x30` | `SF_CTRL_FIFO_STATUS` | FIFO status |
| `0x34` | `SF_CTRL_AES_CFG` | AES encryption config |
| `0x38` | `SF_CTRL_AES_KEY_0` | AES key [31:0] |
| `0x3C` | `SF_CTRL_AES_KEY_1` | AES key [63:32] |
| `0x40` | `SF_CTRL_AES_KEY_2` | AES key [95:64] |
| `0x44` | `SF_CTRL_AES_KEY_3` | AES key [127:96] |
| `0x48` | `SF_CTRL_AES_IV_0` | AES IV [31:0] |
| `0x4C` | `SF_CTRL_AES_IV_1` | AES IV [63:32] |
| `0x50` | `SF_CTRL_XIP_CTRL` | XIP mode control |
| `0x54` | `SF_CTRL_XIP_ADDR` | XIP address configuration |
| `0x58` | `SF_CTRL_XIP_MASK` | XIP address mask |
| `0x5C` | `SF_CTRL_SF2_CTRL` | Second flash controller |

---

## Key Register Fields

### SF_CTRL_CFG (Offset 0x00)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `SF_VALID` | [0] | Config Valid | 1=config is valid |
| `SF_BUSY` | [1] | Controller Busy | 1=controller is busy |
| `SF_IO2` | [2] | IO2 Status | GPIO for quad mode |
| `SF_IO3` | [3] | IO3 Status | GPIO for quad mode |
| `SF_QPI_MODE` | [4] | QPI Mode | 1=enable QPI mode |
| `SF_DTR_MODE` | [5] | DTR Mode | 1=double transfer rate |
| `SF_OPS` | [6] | SPI Operation Size | 0=8-bit, 1=16-bit, 2=32-bit |
| `SF_DUAL` | [7] | Dual Mode | 1=dual flash mode |
| `SF_QWIO` | [8] | QWIO Mode | 1=quad-wide I/O |
| `SF_SEQ_MODE` | [9] | Sequential Mode | 1=enable sequential reads |

### SF_CTRL_CMD (Offset 0x08)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `SF_CMD` | [7:0] | Command Code | Flash command to send |
| `SF_CMD_EN` | [8] | Command Enable | 1=send command |
| `SF_CMD_DUMMY` | [15:9] | Dummy Cycles | Number of dummy cycles |
| `SF_ADDR_EN` | [16] | Address Enable | 1=send address |
| `SF_ADDR_NBYTES` | [20:17] | Address Bytes | Number of address bytes (1-4) |
| `SF_TX_EN` | [21] | TX Enable | 1=enable transmit |
| `SF_RX_EN` | [22] | RX Enable | 1=enable receive |
| `SF_TX_NBYTES` | [29:23] | TX Bytes | Number of TX bytes |
| `SF_RX_NBYTES` | [31:30] | RX Bytes | Number of RX bytes (0-4) |

### SF_CTRL_CTRL (Offset 0x18)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `SF_START` | [0] | Start | Write 1 to start operation |
| `SF_STOP` | [1] | Stop | Write 1 to stop |
| `SF_MEM_IF_EN` | [2] | Memory Interface En | 1=enable XIP mode |
| `SF_REG_IF_EN` | [3] | Register Interface En | 1=enable register mode |
| `SF_INT_MODE` | [4] | Interrupt Mode | 1=poll status, 0=use INT pin |
| `SF_LOCK` | [5] | Lock | Lock flash configuration |

### SF_CTRL_INT_STATUS (Offset 0x1C)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `SF_INT_CMD_DONE` | [0] | Command Done | 1=command completed |
| `SF_INT_RX_FULL` | [1] | RX FIFO Full | 1=receive FIFO full |
| `SF_INT_RX_HALF` | [2] | RX FIFO Half | 1=receive FIFO half full |
| `SF_INT_TX_EMPTY` | [3] | TX FIFO Empty | 1=transmit FIFO empty |
| `SF_INT_TX_HALF` | [4] | TX FIFO Half | 1=transmit FIFO half full |
| `SF_INT_ERR` | [5] | Error | 1=error occurred |

### SF_CTRL_FIFO_STATUS (Offset 0x30)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `SF_RX_CNT` | [5:0] | RX Count | Number of bytes in RX FIFO |
| `SF_TX_CNT` | [13:8] | TX Count | Number of bytes in TX FIFO |
| `SF_RX_FULL` | [16] | RX Full | 1=receive FIFO full |
| `SF_RX_EMPTY` | [17] | RX Empty | 1=receive FIFO empty |
| `SF_TX_FULL` | [18] | TX Full | 1=transmit FIFO full |
| `SF_TX_EMPTY` | [19] | TX Empty | 1=transmit FIFO empty |

### SF_CTRL_TIMING (Offset 0x24)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `SF_TCSS` | [4:0] | CS Setup Time | CS setup time in clock cycles |
| `SF_TCSH` | [9:5] | CS Hold Time | CS hold time in clock cycles |
| `SF_TCKH` | [14:10] | Clock High | Clock high time |
| `SF_TCKL` | [19:15] | Clock Low | Clock low time |
| `SF_CLOCK` | [31:24] | Clock Divider | SPI clock = AHB clock / (2*(n+1)) |

---

## Register-Level Programming Example

```c
#include "bl602.h"
#include "sf_ctrl_reg.h"

#define SF_CTRL_BASE  ((uint32_t)0x4000B000)

/* Read flash JEDEC ID (manufacturer, device ID) */
uint32_t Flash_Read_ID(void)
{
    uint32_t tmpVal;

    /* Configure: 1 byte command, 3 bytes TX, 3 bytes RX */
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CMD, 0x9F);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_TXDATA, 0);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_INT_STATUS, 0x3F);  /* clear ints */

    /* Set up: CMD_EN, TX_EN, RX_EN, 3 bytes each */
    tmpVal = 0;
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_CMD_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_TX_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_RX_EN);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_TX_NBYTES, 0);  /* 1 byte TX */
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_RX_NBYTES, 3);  /* 3 bytes RX */
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_ADDR_NBYTES, 0);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CMD, tmpVal);

    /* Start */
    tmpVal = BL_RD_REG(SF_CTRL_BASE, SF_CTRL_CTRL);
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_START);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CTRL, tmpVal);

    /* Wait for done */
    do {
        tmpVal = BL_RD_REG(SF_CTRL_BASE, SF_CTRL_INT_STATUS);
    } while (!BL_IS_REG_BIT_SET(tmpVal, SF_INT_CMD_DONE));

    /* Read result */
    return BL_RD_REG(SF_CTRL_BASE, SF_CTRL_RXDATA);
}

/* Read 256 bytes from flash using direct read command */
void Flash_Read_Data(uint32_t addr, uint8_t *buf, uint32_t len)
{
    uint32_t tmpVal;
    uint32_t i;

    for (i = 0; i < len; i++) {
        /* Send fast read command (0x0B) with 3-byte addr, 1 dummy byte */
        tmpVal = 0;
        tmpVal = BL_SET_REG_BIT(tmpVal, SF_CMD_EN);
        tmpVal = BL_SET_REG_BIT(tmpVal, SF_ADDR_EN);
        tmpVal = BL_SET_REG_BIT(tmpVal, SF_TX_EN);
        tmpVal = BL_SET_REG_BIT(tmpVal, SF_RX_EN);
        tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_CMD, 0x0B);     /* fast read */
        tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_CMD_DUMMY, 1);  /* 1 dummy cycle */
        tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_ADDR_NBYTES, 3); /* 3 byte addr */
        tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_TX_NBYTES, 0);
        tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_RX_NBYTES, 1);  /* 1 byte RX */
        BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CMD, tmpVal);

        /* Set address */
        BL_WR_REG(SF_CTRL_BASE, SF_CTRL_ADDR, addr + i);

        /* Start */
        tmpVal = BL_RD_REG(SF_CTRL_BASE, SF_CTRL_CTRL);
        tmpVal = BL_SET_REG_BIT(tmpVal, SF_START);
        BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CTRL, tmpVal);

        /* Wait and read */
        do {
            tmpVal = BL_RD_REG(SF_CTRL_BASE, SF_CTRL_INT_STATUS);
        } while (!BL_IS_REG_BIT_SET(tmpVal, SF_INT_CMD_DONE));

        buf[i] = (uint8_t)BL_RD_REG(SF_CTRL_BASE, SF_CTRL_RXDATA);
    }
}

/* Write enable and erase sector (4KB) */
void Flash_Erase_Sector(uint32_t addr)
{
    uint32_t tmpVal;

    /* Write enable */
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CMD, 0x06);
    tmpVal = 0;
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_CMD_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_TX_EN);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_TX_NBYTES, 0);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CMD, tmpVal);
    tmpVal = BL_RD_REG(SF_CTRL_BASE, SF_CTRL_CTRL);
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_START);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CTRL, tmpVal);

    do {
        tmpVal = BL_RD_REG(SF_CTRL_BASE, SF_CTRL_INT_STATUS);
    } while (!BL_IS_REG_BIT_SET(tmpVal, SF_INT_CMD_DONE));

    /* Sector erase 0x20 (4KB) */
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_ADDR, addr);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CMD, 0x20);
    tmpVal = 0;
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_CMD_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_ADDR_EN);
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_TX_EN);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_ADDR_NBYTES, 3);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, SF_TX_NBYTES, 0);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CMD, tmpVal);
    tmpVal = BL_RD_REG(SF_CTRL_BASE, SF_CTRL_CTRL);
    tmpVal = BL_SET_REG_BIT(tmpVal, SF_START);
    BL_WR_REG(SF_CTRL_BASE, SF_CTRL_CTRL, tmpVal);

    /* Wait for erase to complete */
    do {
        tmpVal = BL_RD_REG(SF_CTRL_BASE, SF_CTRL_CFG);
    } while (BL_IS_REG_BIT_SET(tmpVal, SF_BUSY));
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_flash.h`

```c
/* HOSAL Flash device configuration */
typedef struct {
    uint8_t       port;       /* flash port (0) */
    uint32_t      flash_type; /* flash type (unused for now) */
    uint32_t      block_size; /* erase block size */
    uint32_t      sector_size; /* erase sector size */
    uint32_t      page_size;  /* write page size */
    void         *priv;        /* private data */
} hosal_flash_dev_t;

/* Flash geometry - obtained from JEDEC ID */
typedef struct {
    uint32_t jedec_id;      /* manufacturer + device ID */
    uint32_t capacity;       /* flash capacity in bytes */
    uint32_t block_size;     /* erase block size */
    uint32_t sector_size;    /* erase sector size */
    uint32_t page_size;      /* write page size */
} hosal_flash_info_t;

/* HOSAL Flash operations */
enum {
    HOSAL_FLASH_READ = 0,
    HOSAL_FLASH_WRITE = 1,
    HOSAL_FLASH_ERASE = 2,
};

/**
 * @brief Initialize flash device
 * @param dev  flash device pointer
 * @return 0 on success, -1 on failure
 */
int hosal_flash_init(hosal_flash_dev_t *dev);

/**
 * @brief Read from flash
 * @param dev  flash device pointer
 * @param addr  flash address
 * @param buf  data buffer
 * @param len  number of bytes to read
 * @return 0 on success, -1 on failure
 */
int hosal_flash_read(hosal_flash_dev_t *dev, uint32_t addr, void *buf, uint32_t len);

/**
 * @brief Write to flash (page program)
 * @param dev  flash device pointer
 * @param addr  flash address (must be page-aligned)
 * @param buf  data buffer
 * @param len  number of bytes to write
 * @return 0 on success, -1 on failure
 */
int hosal_flash_write(hosal_flash_dev_t *dev, uint32_t addr, const void *buf, uint32_t len);

/**
 * @brief Erase flash sector
 * @param dev  flash device pointer
 * @param addr  sector address
 * @param len  length to erase (must be sector-aligned)
 * @return 0 on success, -1 on failure
 */
int hosal_flash_erase(hosal_flash_dev_t *dev, uint32_t addr, uint32_t len);

/**
 * @brief Get flash information
 * @param dev  flash device pointer
 * @param info  flash info structure
 * @return 0 on success, -1 on failure
 */
int hosal_flash_get_info(hosal_flash_dev_t *dev, hosal_flash_info_t *info);

/**
 * @brief Deinitialize flash device
 * @param dev  flash device pointer
 * @return 0 on success, -1 on failure
 */
int hosal_flash_finalize(hosal_flash_dev_t *dev);
```

### HOSAL Flash Usage Example

```c
#include "hosal_flash.h"

hosal_flash_dev_t flash_dev;

void flash_example(void)
{
    uint8_t read_buf[256];
    uint8_t write_buf[256];
    int ret;
    hosal_flash_info_t info;

    /* Initialize */
    ret = hosal_flash_init(&flash_dev);
    if (ret != 0) {
        printf("Flash init failed\r\n");
        return;
    }

    /* Get flash info */
    ret = hosal_flash_get_info(&flash_dev, &info);
    if (ret == 0) {
        printf("Flash JEDEC ID: 0x%08X, Size: %u bytes\r\n",
               info.jedec_id, info.capacity);
    }

    /* Read first 256 bytes */
    ret = hosal_flash_read(&flash_dev, 0, read_buf, sizeof(read_buf));
    if (ret == 0) {
        printf("Read successful\r\n");
    }

    /* Write page (must be page-aligned, typically 256 bytes) */
    for (int i = 0; i < 256; i++) {
        write_buf[i] = i;
    }
    ret = hosal_flash_write(&flash_dev, 0x1000, write_buf, 256);
    if (ret == 0) {
        printf("Write successful\r\n");
    }

    /* Erase 4KB sector */
    ret = hosal_flash_erase(&flash_dev, 0x1000, 4096);
    if (ret == 0) {
        printf("Erase successful\r\n");
    }

    hosal_flash_finalize(&flash_dev);
}
```

### Notes

- The BL602 supports external SPI flash up to 16MB via the SF controller
- Flash is memory-mapped at `0x23000000` for XIP operations (read-while-execute)
- Common flash commands are handled automatically; use HOSAL for standard operations
- For encrypted XIP, configure AES keys in the SF_CTRL_AES_* registers
- Flash timing must be configured based on the flash speed grade (typically 80MHz max)
- Always perform write enable (0x06) before any write or erase operation
