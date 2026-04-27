# Camera (bl_cam) - BL702 Camera Driver

## Overview

bl_cam is the camera driver for BL702 embedded processors. It provides an interface to configure camera sensors, capture frames in various formats (JPEG, YUV), and manage the camera subsystem.

## Location

```
/home/seahi/workspase/Ai-Thinker-WB2/components/platform/hosal/bl702_hal/bl_cam.c
/home/seahi/workspase/Ai-Thinker-WB2/components/platform/hosal/bl702_hal/bl_cam.h
```

## Key Files

- `bl_cam.h` - Header with all type definitions and API declarations
- `bl_cam.c` - Full implementation

## Dependencies

Requires `sensor.h` for camera description types.

---

## Core API

### Configuration

```c
int bl_cam_config_get(uint8_t *quality, uint16_t *width, uint16_t *height);
```

Get current camera configuration.

**Parameters:**
- `quality` - Output: JPEG quality setting
- `width` - Output: Image width
- `height` - Output: Image height

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_config_update(uint8_t quality);
```

Update camera JPEG quality setting.

**Parameters:**
- `quality` - JPEG encoding quality (typically 1-100)

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_init(int enable_mjpeg, const rt_camera_desc *desc);
```

Initialize the camera subsystem.

**Parameters:**
- `enable_mjpeg` - Enable MJPEG encoding (1) or YUV output (0)
- `desc` - Camera descriptor with sensor configuration

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_restart(int enable_mjpeg);
```

Restart camera with new MJPEG setting.

**Parameters:**
- `enable_mjpeg` - Enable MJPEG encoding (1) or YUV output (0)

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_enable_24MRef(void);
```

Enable 24MHz camera reference clock.

**Returns:** 0 on success, negative errno on error

---

### Frame Capture

```c
int bl_cam_frame_get(uint32_t *frames, uint8_t **ptr1, uint32_t *len1, 
                     uint8_t **ptr2, uint32_t *len2);
```

Get the next available camera frame (double-buffered).

**Parameters:**
- `frames` - Output: Frame counter value
- `ptr1` - Output: Pointer to first buffer
- `len1` - Output: Length of first buffer
- `ptr2` - Output: Pointer to second buffer (may be NULL)
- `len2` - Output: Length of second buffer

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_frame_fifo_get(uint32_t *frames, uint8_t **ptr1, uint32_t *len1, 
                          uint8_t **ptr2, uint32_t *len2);
```

Get frame from camera FIFO (alternative method).

**Parameters:** Same as `bl_cam_frame_get`

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_frame_pop(void);
```

Pop (release) the current frame after processing.

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_frame_pop_old(void);
```

Pop the oldest frame if multiple frames are queued.

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_frame_wait(void);
```

Wait for next frame to be available.

**Returns:** 0 on success, negative errno on error

---

### YUV Frame Operations

```c
int bl_cam_yuv_frame_wait(void);
```

Wait for next YUV frame to be available.

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_yuv_frame_get(uint32_t *frames, uint8_t **ptr1, uint32_t *len1, 
                         uint8_t **ptr2, uint32_t *len2);
```

Get next YUV frame (double-buffered).

**Parameters:** Same as `bl_cam_frame_get`

**Returns:** 0 on success, negative errno on error

---

```c
int bl_cam_yuv_frame_pop(void);
```

Pop (release) the current YUV frame after processing.

**Returns:** 0 on success, negative errno on error

---

### MJPEG Encoding

```c
int bl_cam_mjpeg_encoder(uint32_t yuv_addr, uint32_t jpeg_addr, uint32_t *jpeg_size, 
                         uint32_t width, uint32_t height, uint32_t quality);
```

Encode YUV image to JPEG/MJPEG format using hardware encoder.

**Parameters:**
- `yuv_addr` - Source YUV image address
- `jpeg_addr` - Destination JPEG buffer address
- `jpeg_size` - Output: Encoded JPEG size
- `width` - Image width
- `height` - Image height
- `quality` - JPEG quality (1-100)

**Returns:** 0 on success, negative errno on error

---

## Usage Example

```c
#include "bl_cam.h"

// Initialize camera in MJPEG mode
rt_camera_desc desc = {
    .width = 640,
    .height = 480,
    // ... other sensor settings
};

int ret = bl_cam_init(1, &desc);
if (ret != 0) {
    // Handle error
}

// Capture frames
while (1) {
    bl_cam_frame_wait();
    
    uint32_t frames;
    uint8_t *ptr1, *ptr2;
    uint32_t len1, len2;
    
    bl_cam_frame_get(&frames, &ptr1, &len1, &ptr2, &len2);
    
    // Process frame data...
    
    bl_cam_frame_pop();
}
```

## Embedded Considerations

- Frame buffers are typically DMA-accessible memory
- MJPEG encoding offloads CPU compared to software JPEG encoding
- Frame timing is critical for video applications
- Sensor configuration depends on the specific camera module used
