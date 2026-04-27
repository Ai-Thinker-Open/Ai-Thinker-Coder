# BL602 DMA Register-Level API Reference

**Source file:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/dma_reg.h`

**Base Address:** `0x4000C000`

---

## Register Overview

### Global DMA Registers

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x00` | DMA_IntStatus | Interrupt Status Register |
| `0x04` | DMA_IntTCStatus | Transfer Complete Interrupt Status |
| `0x08` | DMA_IntTCClear | Transfer Complete Interrupt Clear |
| `0x0C` | DMA_IntErrorStatus | Error Interrupt Status |
| `0x10` | DMA_IntErrClr | Error Interrupt Clear |
| `0x14` | DMA_RawIntTCStatus | Raw Transfer Complete Status |
| `0x18` | DMA_RawIntErrorStatus | Raw Error Status |
| `0x1C` | DMA_EnbldChns | Enabled Channels Register |
| `0x20` | DMA_SoftBReq | Software Big Request Register |
| `0x24` | DMA_SoftSReq | Software Single Request Register |
| `0x28` | DMA_SoftLBReq | Software Last Big Request Register |
| `0x2C` | DMA_SoftLSReq | Software Last Single Request Register |
| `0x30` | DMA_Top_Config | Top Configuration Register |
| `0x34` | DMA_Sync | DMA Request Synchronization |

### Channel 0 Registers (Offset base 0x100)

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x100` | DMA_C0SrcAddr | Channel 0 Source Address |
| `0x104` | DMA_C0DstAddr | Channel 0 Destination Address |
| `0x108` | DMA_C0LLI | Channel 0 Linked List Item |
| `0x10C` | DMA_C0Control | Channel 0 Control |
| `0x110` | DMA_C0Config | Channel 0 Configuration |

### Channel 1 Registers (Offset base 0x200)

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x200` | DMA_C1SrcAddr | Channel 1 Source Address |
| `0x204` | DMA_C1DstAddr | Channel 1 Destination Address |
| `0x208` | DMA_C1LLI | Channel 1 Linked List Item |
| `0x20C` | DMA_C1Control | Channel 1 Control |
| `0x210` | DMA_C1Config | Channel 1 Configuration |

### Channel 2 Registers (Offset base 0x300)

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x300` | DMA_C2SrcAddr | Channel 2 Source Address |
| `0x304` | DMA_C2DstAddr | Channel 2 Destination Address |
| `0x308` | DMA_C2LLI | Channel 2 Linked List Item |
| `0x30C` | DMA_C2Control | Channel 2 Control |
| `0x310` | DMA_C2Config | Channel 2 Configuration |

---

## Key Register Fields

### DMA_Top_Config - Top Configuration (Offset 0x30)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| E | [0] | R/W | Master enable bit for DMA controller |
| M | [1] | R/W | Memory endianness configuration |

### DMA_CxControl - Channel Control (Offsets 0x10C, 0x20C, 0x30C, ...)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TRANSFERSIZE | [11:0] | R/W | Number of transfers (1-4096) |
| SBSIZE | [14:12] | R/W | Source burst size (0=1, 1=2, 2=4, 3=8, 4=16) |
| DBSIZE | [17:15] | R/W | Destination burst size |
| SWIDTH | [20:18] | R/W | Source transfer width (0=8-bit, 1=16-bit, 2=32-bit, 3=64-bit) |
| DWIDTH | [23:21] | R/W | Destination transfer width |
| SLARGERD | [24] | R/W | Source larger than destination enable |
| SI | [26] | R/W | Source increment (1=increment, 0=fixed) |
| DI | [27] | R/W | Destination increment (1=increment, 0=fixed) |
| PROT | [30:28] | R/W | Protection control |
| I | [31] | R/W | Interrupt enable on transfer complete |

### DMA_CxConfig - Channel Configuration (Offsets 0x110, 0x210, 0x310, ...)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| E | [0] | R/W | Channel enable |
| SRCPERIPHERAL | [5:1] | R/W | Source peripheral select (0-31) |
| DSTPERIPHERAL | [10:6] | R/W | Destination peripheral select (0-31) |
| FLOWCNTRL | [13:11] | R/W | Flow control (0=M2M, 1=M2P, 2=P2M, 3=P2P, 4=LIM) |
| IE | [14] | R/W | Error interrupt enable |
| ITC | [15] | R/W | Terminal count interrupt enable |
| L | [16] | R/W | Last descriptor (for linked lists) |
| A | [17] | R/W | Active flag (read-only status) |
| H | [18] | R/W | Halt flag |
| LLICOUNTER | [29:20] | R/W | Linked list repeat count |

### DMA_LLI - Linked List Item (Offsets 0x108, 0x208, 0x308, ...)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| LLI | [31:2] | R/W | Next LLI address (word-aligned) |
| Reserved | [1:0] | - | Must be zero |

### DMA_SoftBReq / DMA_SoftSReq / DMA_SoftLBReq / DMA_SoftLSReq (Offsets 0x20-0x2C)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| SOFTBREQ | [31:0] | R/W | Per-peripheral software request bits |

---

## Register-Level Programming Example

### Memory-to-Memory DMA Transfer Example

```c
#include "bl602.h"
#include "bl602_dma.h"

#define DMA_BASE 0x4000C000

/* DMA channel configuration structure */
typedef struct {
    uint32_t src_addr;
    uint32_t dst_addr;
    uint32_t transfer_size;
    uint8_t  src_width;   /* 0=8-bit, 1=16-bit, 2=32-bit */
    uint8_t  dst_width;
    uint8_t  src_inc;     /* 1=increment, 0=fixed */
    uint8_t  dst_inc;
    uint8_t  burst_size;  /* 0=1, 1=2, 2=4, 3=8, 4=16 */
} dma_xfer_config_t;

/* Start DMA channel 0 for memory-to-memory transfer */
int dma_m2m_start(uint32_t src, uint32_t dst, uint32_t size)
{
    volatile dma_reg_t *dma = (volatile dma_reg_t *)DMA_BASE;

    /* Enable DMA controller if not already enabled */
    if (!dma->DMA_Top_Config.BF.e) {
        dma->DMA_Top_Config.BF.e = 1;
    }

    /* Stop channel 0 if running */
    dma->DMA_C0Config.BF.e = 0;

    /* Clear any pending interrupts */
    dma->DMA_IntTCClear.WORD = (1 << 0);
    dma->DMA_IntErrClr.WORD = (1 << 0);

    /* Set source address */
    dma->DMA_C0SrcAddr.WORD = src;

    /* Set destination address */
    dma->DMA_C0DstAddr.WORD = dst;

    /* Configure transfer:
     * - 32-bit transfers
     * - Increment both source and destination
     * - Burst size of 4
     * - Enable interrupt on complete
     */
    dma->DMA_C0Control.WORD = 0;
    dma->DMA_C0Control.BF.transfer_size = size;
    dma->DMA_C0Control.BF.sbsize = 2;     /* Burst 4 */
    dma->DMA_C0Control.BF.dbsize = 2;     /* Burst 4 */
    dma->DMA_C0Control.BF.swidth = 2;     /* 32-bit */
    dma->DMA_C0Control.BF.dwidth = 2;     /* 32-bit */
    dma->DMA_C0Control.BF.si = 1;          /* Increment source */
    dma->DMA_C0Control.BF.di = 1;          /* Increment dest */
    dma->DMA_C0Control.BF.i = 1;           /* Interrupt enable */

    /* Configure channel:
     * - Memory to memory transfer
     * - No peripheral select
     * - Enable channel
     */
    dma->DMA_C0Config.WORD = 0;
    dma->DMA_C0Config.BF.flowcntrl = 0;    /* M2M */
    dma->DMA_C0Config.BF.srcperipheral = 0;
    dma->DMA_C0Config.BF.dstperipheral = 0;
    dma->DMA_C0Config.BF.itc = 1;          /* Terminal count interrupt */
    dma->DMA_C0Config.BF.ie = 1;           /* Error interrupt */
    dma->DMA_C0Config.BF.e = 1;            /* Enable channel */

    return 0;
}

/* DMA interrupt handler */
void dma_irq_handler(void)
{
    volatile dma_reg_t *dma = (volatile dma_reg_t *)DMA_BASE;

    /* Check channel 0 transfer complete */
    if (dma->DMA_IntTCStatus.WORD & (1 << 0)) {
        /* Clear interrupt */
        dma->DMA_IntTCClear.WORD = (1 << 0);

        /* Transfer complete - handle callback */
        // ...
    }

    /* Check for errors */
    if (dma->DMA_IntErrorStatus.WORD & (1 << 0)) {
        /* Clear error */
        dma->DMA_IntErrClr.WORD = (1 << 0);

        /* Handle error */
        // ...
    }
}
```

### Peripheral-to-Memory DMA Example (UART)

```c
/* Configure DMA for UART receive (peripheral to memory) */
int dma_uart_rx_start(uint8_t channel, uint32_t uart_rx_addr, uint8_t *buf, uint32_t size)
{
    volatile dma_reg_t *dma = (volatile dma_reg_t *)DMA_BASE;
    uint32_t channel_base = 0x100 + (channel * 0x100);
    uint32_t config_reg = 0x110 + (channel * 0x100);

    /* Enable DMA controller */
    dma->DMA_Top_Config.BF.e = 1;

    /* Stop channel */
    *(volatile uint32_t *)(DMA_BASE + config_reg) &= ~1;

    /* Clear interrupts */
    dma->DMA_IntTCClear.WORD = (1 << channel);
    dma->DMA_IntErrClr.WORD = (1 << channel);

    /* Set source (UART RX register) */
    *(volatile uint32_t *)(DMA_BASE + channel_base) = uart_rx_addr;

    /* Set destination (memory buffer) */
    *(volatile uint32_t *)(DMA_BASE + channel_base + 4) = (uint32_t)buf;

    /* Configure control:
     * - 8-bit transfers
     * - Source fixed (peripheral), dest increment (memory)
     * - Size = number of bytes
     */
    *(volatile uint32_t *)(DMA_BASE + channel_base + 8) =
        (size & 0xFFF) |          /* TransferSize */
        (0 << 12) |               /* SBSIZE = 1 */
        (0 << 15) |               /* DBSIZE = 1 */
        (0 << 18) |               /* SWIDTH = 8-bit */
        (0 << 21) |               /* DWIDTH = 8-bit */
        (0 << 26) |               /* SI = 0 (fixed) */
        (1 << 27) |               /* DI = 1 (increment) */
        (1 << 31);                /* I = 1 (interrupt) */

    /* Configure channel:
     * - Peripheral to Memory
     * - UART peripheral as source
     * - Flow controlled by peripheral
     */
    *(volatile uint32_t *)(DMA_BASE + config_reg) =
        (0 << 0) |                /* E = 0 for now */
        (17 << 1) |              /* SRCPERIPHERAL = UART (peripheral #17) */
        (0 << 6) |               /* DSTPERIPHERAL = 0 (memory) */
        (2 << 11) |              /* FLOWCNTRL = P2M */
        (1 << 14) |               /* IE = 1 */
        (1 << 15);                /* ITC = 1 */

    /* Enable channel */
    *(volatile uint32_t *)(DMA_BASE + config_reg) |= 1;

    return 0;
}
```

### Linked List Transfer Example

```c
/* Linked list item structure */
typedef struct {
    uint32_t src_addr;      /* Source address */
    uint32_t dst_addr;      /* Destination address */
    uint32_t next_lli;       /* Next LLI address (0 = end of list) */
    uint32_t control;        /* Transfer control word */
} dma_lli_t;

/* Execute linked list transfer */
int dma_lli_start(dma_lli_t *lli_list, uint32_t num_items)
{
    volatile dma_reg_t *dma = (volatile dma_reg_t *)DMA_BASE;
    dma_lli_t *p_lli = lli_list;

    /* Configure each LLI in the list */
    for (uint32_t i = 0; i < num_items; i++) {
        uint32_t ch_base = 0x100 + (i * 0x100);

        /* Set source address */
        *(volatile uint32_t *)(DMA_BASE + ch_base) = p_lli->src_addr;

        /* Set destination address */
        *(volatile uint32_t *)(DMA_BASE + ch_base + 4) = p_lli->dst_addr;

        /* Set next LLI (must be word-aligned, lower 2 bits = 0) */
        *(volatile uint32_t *)(DMA_BASE + ch_base + 8) = 
            (i < num_items - 1) ? ((uint32_t)(p_lli + 1) & ~0x3) : 0;

        /* Set control */
        *(volatile uint32_t *)(DMA_BASE + ch_base + 12) = p_lli->control;

        /* Configure channel */
        uint32_t config_reg = 0x110 + (i * 0x100);
        *(volatile uint32_t *)(DMA_BASE + config_reg) =
            (0 << 0) |             /* E = 0 for now */
            (0 << 1) |             /* SRCPERIPHERAL = 0 */
            (0 << 6) |             /* DSTPERIPHERAL = 0 */
            (0 << 11) |            /* FLOWCNTRL = M2M */
            (1 << 14) |            /* IE */
            (1 << 15) |            /* ITC */
            ((i < num_items - 1) ? (1 << 16) : 0);  /* L (not last if more items) */

        p_lli++;
    }

    /* Enable first channel to start transfer */
    dma->DMA_C0Config.BF.e = 1;

    return 0;
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_dma.h`

### Data Types and Constants

```c
/* Interrupt flags */
#define HOSAL_DMA_INT_TRANS_COMPLETE  0  /* Transfer complete interrupt */
#define HOSAL_DMA_INT_TRANS_ERROR     1  /* Transfer error interrupt */

typedef void (*hosal_dma_irq_t)(void *p_arg, uint32_t flag);

struct hosal_dma_chan {
    uint8_t used;
    hosal_dma_irq_t callback;
    void *p_arg;
};

typedef struct hosal_dma_dev {
    int max_chans;
    struct hosal_dma_chan *used_chan;
    void *priv;
} hosal_dma_dev_t;

typedef int hosal_dma_chan_t;
```

### API Functions

| Function | Description |
|----------|-------------|
| `int hosal_dma_init(void)` | Initialize DMA controller |
| `hosal_dma_chan_t hosal_dma_chan_request(int flag)` | Request a DMA channel |
| `int hosal_dma_chan_release(hosal_dma_chan_t chan)` | Release a DMA channel |
| `int hosal_dma_chan_start(hosal_dma_chan_t chan)` | Start DMA transfer |
| `int hosal_dma_chan_stop(hosal_dma_chan_t chan)` | Stop DMA transfer |
| `int hosal_dma_irq_callback_set(hosal_dma_chan_t chan, hosal_dma_irq_t pfn, void *p_arg)` | Set DMA callback |
| `int hosal_dma_finalize(void)` | De-initialize DMA |

### HOSAL Usage Example

```c
#include "hosal_dma.h"

hosal_dma_chan_t dma_chan;
volatile uint8_t src_buffer[256];
volatile uint8_t dst_buffer[256];
volatile int dma_complete = 0;

void dma_callback(void *p_arg, uint32_t flag)
{
    if (flag == HOSAL_DMA_INT_TRANS_COMPLETE) {
        dma_complete = 1;
        /* Handle transfer completion */
    } else if (flag == HOSAL_DMA_INT_TRANS_ERROR) {
        /* Handle transfer error */
    }
}

void app_dma_init(void)
{
    /* Initialize DMA controller */
    hosal_dma_init();

    /* Request a DMA channel */
    dma_chan = hosal_dma_chan_request(0);
    if (dma_chan < 0) {
        /* Handle error */
        return;
    }

    /* Set callback */
    hosal_dma_irq_callback_set(dma_chan, dma_callback, NULL);
}

void app_dma_start_transfer(void)
{
    /* Note: HOSAL DMA abstract interface - actual source/dest/size
     * configuration depends on underlying implementation */
    hosal_dma_chan_start(dma_chan);
}

void app_dma_stop(void)
{
    hosal_dma_chan_stop(dma_chan);
    hosal_dma_chan_release(dma_chan);
    hosal_dma_finalize();
}
```
