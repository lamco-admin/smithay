# Buffer Validation Analysis for ext-image-copy-capture-v1

**Author:** Generated for Smithay upstream contribution
**Date:** 2026-01-10
**Status:** Ready for implementation

---

## Executive Summary

Our current `ext-image-copy-capture-v1` implementation is missing buffer validation before invoking the compositor's `frame()` callback. This document analyzes how other compositors handle buffer validation, examines the protocol requirements, and recommends an implementation approach that is robust, flexible, and consistent with Smithay's patterns.

**Key Finding:** Buffer validation MUST occur before the `frame()` callback, with validation failures returning `FailureReason::BufferConstraints`.

---

## Table of Contents

1. [Protocol Requirements](#1-protocol-requirements)
2. [Current Implementation Gap](#2-current-implementation-gap)
3. [Compositor Comparison](#3-compositor-comparison)
4. [Smithay Buffer Utilities](#4-smithay-buffer-utilities)
5. [Design Options](#5-design-options)
6. [Recommended Implementation](#6-recommended-implementation)
7. [Implementation Plan](#7-implementation-plan)

---

## 1. Protocol Requirements

From the `ext-image-copy-capture-v1` protocol specification:

### 1.1 Buffer Constraints Advertisement

The compositor advertises buffer requirements via:
- `buffer_size(width, height)` - exactly one, required
- `shm_format(format)` - zero or more
- `dmabuf_device(device)` - zero or one
- `dmabuf_format(format, modifiers)` - zero or more
- `done()` - signals end of constraints

### 1.2 attach_buffer Requirements

> "The buffer must match the dimensions specified in the `buffer_size` event."

### 1.3 Capture Failure Reasons

The `failed` event includes a `reason` enum:

| Reason | Description |
|--------|-------------|
| `buffer_constraints` | Buffer doesn't match latest session constraints |
| `stopped` | Session is no longer available |
| `unknown` | Unspecified runtime error |

### 1.4 Protocol Expectation

The specification clearly states that a buffer not matching constraints should result in `buffer_constraints` failure. This is NOT a protocol error (which would kill the client) but a graceful failure that allows the client to retry.

---

## 2. Current Implementation Gap

### 2.1 What We Currently Validate

In `src/wayland/image_copy_capture/mod.rs` at the `Capture` request handler:

```rust
// Lines 1182-1191
if inner.buffer.is_none() {
    inner.fail(resource, FailureReason::BufferConstraints);
    return;
}
```

**Current validation:**
- Buffer presence: YES
- Buffer size: NO
- SHM format: NO
- DMA-BUF format: NO
- DMA-BUF modifier: NO

### 2.2 Impact

Without validation, the compositor's `frame()` callback receives potentially invalid buffers. This could cause:
- Rendering artifacts
- Buffer overflows
- Crashes in the rendering pipeline
- Security vulnerabilities from untrusted client data

---

## 3. Compositor Comparison

### 3.1 wlroots (Reference Implementation)

**Location:** `types/wlr_ext_image_copy_capture_v1.c`

**Validation approach:** Strict exact-match

```c
// Size validation - exact match required
if (src->width != dst->width || src->height != dst->height) {
    wlr_ext_image_copy_capture_frame_v1_fail(frame,
        EXT_IMAGE_COPY_CAPTURE_FRAME_V1_FAILURE_REASON_BUFFER_CONSTRAINTS);
    return false;
}
```

**Format validation:**
- Checks `frame->session->source->dmabuf_formats.len > 0`
- Checks `frame->session->source->shm_formats_len > 0`
- Fails if no supported formats

**Key characteristic:** Validation happens during copy operation.

### 3.2 COSMIC Compositor

**Location:** `src/wayland/protocols/screencopy.rs`

**Validation approach:** Minimum size with format matching

```rust
// Size validation - allows larger buffers
if buffer_size.w < constraints.size.w || buffer_size.h < constraints.size.h {
    // Fail with BufferConstraints
}

// SHM format validation
if !constraints.shm.contains(&buffer_data.format) {
    // Fail with BufferConstraints
}

// DMA-BUF format + modifier validation
if !formats.iter().any(|(f, m)| *f == dmabuf.format().code && m.contains(&modifier)) {
    // Fail with BufferConstraints
}
```

**Key characteristics:**
- Validation at `Capture` request time
- Allows buffers larger than constraints (more permissive)
- Validates format AND modifier for DMA-BUF

### 3.3 Jay Compositor

**Validation approach:** Similar to COSMIC

- Size validation (minimum match)
- Format in allowed list
- Modifier validation for DMA-BUF

### 3.4 Niri Compositor

Uses cosmic-comp's protocols crate, inherits same validation approach.

### 3.5 Summary Table

| Compositor | Size Check | Format Check | Modifier Check | When Validated |
|------------|------------|--------------|----------------|----------------|
| wlroots | Exact match | Yes | Yes | During copy |
| COSMIC | Minimum size | Yes | Yes | At Capture |
| Jay | Minimum size | Yes | Yes | At Capture |
| Niri | (via cosmic) | (via cosmic) | (via cosmic) | At Capture |
| **Ours** | **None** | **None** | **None** | **N/A** |

---

## 4. Smithay Buffer Utilities

### 4.1 Buffer Type Detection

```rust
// src/backend/renderer/mod.rs
pub fn buffer_type(buffer: &WlBuffer) -> Option<BufferType>

pub enum BufferType {
    Shm,
    Dma,
    Egl,  // with backend_egl + use_system_lib
    SinglePixel,
}
```

### 4.2 SHM Buffer Access

```rust
// src/wayland/shm/mod.rs
pub fn with_buffer_contents<F, T>(
    buffer: &WlBuffer,
    f: F
) -> Result<T, BufferAccessError>
where
    F: FnOnce(*const u8, usize, BufferData) -> T

pub struct BufferData {
    pub offset: i32,
    pub width: i32,
    pub height: i32,
    pub stride: i32,
    pub format: wl_shm::Format,
}
```

### 4.3 DMA-BUF Access

```rust
// src/wayland/dmabuf/mod.rs
pub fn get_dmabuf(buffer: &WlBuffer) -> Result<&Dmabuf, UnmanagedResource>

// src/backend/allocator/dmabuf.rs
impl Buffer for Dmabuf {
    fn size(&self) -> Size<i32, BufferCoords>
    fn format(&self) -> Format  // { code: Fourcc, modifier: Modifier }
}

// With backend_drm feature
impl Dmabuf {
    pub fn node(&self) -> Option<DrmNode>
}
```

### 4.4 Required Imports

```rust
use crate::backend::renderer::{buffer_type, BufferType};
use crate::wayland::shm::{with_buffer_contents, BufferData};
use crate::wayland::dmabuf::get_dmabuf;
use crate::backend::allocator::Buffer;  // for .size() and .format()
```

---

## 5. Design Options

### 5.1 Option A: Validation in Library (Recommended)

Validate buffers within the `Dispatch` implementation before calling `frame()`.

**Pros:**
- Compositors get validated buffers automatically
- Consistent behavior across all Smithay-based compositors
- Reduces boilerplate in compositor code
- Security boundary at the library level

**Cons:**
- Less flexibility for compositors wanting custom validation
- All compositors must accept the validation rules

### 5.2 Option B: Validation via Callback

Add a `validate_buffer()` callback to `ImageCopyCaptureHandler`.

```rust
fn validate_buffer(
    &mut self,
    session: &SessionRef,
    buffer: &WlBuffer,
    constraints: &BufferConstraints,
) -> Result<(), FailureReason> {
    // Default implementation with standard validation
}
```

**Pros:**
- Compositors can override validation logic
- Maximum flexibility

**Cons:**
- More API surface
- Risk of compositors skipping validation
- Complexity

### 5.3 Option C: Two-Phase Frame Handling

```rust
fn prepare_frame(&mut self, session: &SessionRef, frame: &FrameRef) -> FrameResult;
fn execute_frame(&mut self, session: &SessionRef, frame: Frame, buffer: ValidatedBuffer);
```

**Pros:**
- Clear separation of validation and execution
- Type-safe "validated buffer" concept

**Cons:**
- Significant API change
- Overengineered for the use case

### 5.4 Recommendation: Option A with Escape Hatch

Implement validation in the library (Option A), but provide access to the raw buffer so compositors can perform additional validation if needed:

```rust
impl Frame {
    pub fn buffer(&self) -> &WlBuffer { ... }  // Already exists via FrameRef
}
```

---

## 6. Recommended Implementation

### 6.1 Validation Logic

```rust
fn validate_buffer(
    buffer: &WlBuffer,
    constraints: &BufferConstraints,
) -> Result<(), FailureReason> {
    match buffer_type(buffer) {
        Some(BufferType::Shm) => validate_shm_buffer(buffer, constraints),
        Some(BufferType::Dma) => validate_dmabuf(buffer, constraints),
        _ => Err(FailureReason::BufferConstraints),
    }
}

fn validate_shm_buffer(
    buffer: &WlBuffer,
    constraints: &BufferConstraints,
) -> Result<(), FailureReason> {
    with_buffer_contents(buffer, |_, _, data| {
        // Size check: buffer must be at least as large as constraints
        if data.width < constraints.size.w || data.height < constraints.size.h {
            return Err(FailureReason::BufferConstraints);
        }

        // Format check: must be in allowed list
        if !constraints.shm.contains(&data.format) {
            return Err(FailureReason::BufferConstraints);
        }

        Ok(())
    }).map_err(|_| FailureReason::BufferConstraints)?
}

#[cfg(feature = "backend_drm")]
fn validate_dmabuf(
    buffer: &WlBuffer,
    constraints: &BufferConstraints,
) -> Result<(), FailureReason> {
    let dmabuf = get_dmabuf(buffer)
        .map_err(|_| FailureReason::BufferConstraints)?;

    let dma_constraints = constraints.dma.as_ref()
        .ok_or(FailureReason::BufferConstraints)?;

    let size = dmabuf.size();

    // Size check
    if size.w < constraints.size.w || size.h < constraints.size.h {
        return Err(FailureReason::BufferConstraints);
    }

    // Format + modifier check
    let format = dmabuf.format();
    let format_valid = dma_constraints.formats.iter().any(|(f, modifiers)| {
        *f == format.code && modifiers.contains(&format.modifier)
    });

    if !format_valid {
        return Err(FailureReason::BufferConstraints);
    }

    Ok(())
}

#[cfg(not(feature = "backend_drm"))]
fn validate_dmabuf(
    _buffer: &WlBuffer,
    _constraints: &BufferConstraints,
) -> Result<(), FailureReason> {
    // No DMA-BUF support without backend_drm
    Err(FailureReason::BufferConstraints)
}
```

### 6.2 Integration Point

In the `Capture` request handler, after checking buffer presence:

```rust
ext_image_copy_capture_frame_v1::Request::Capture => {
    // ... existing checks ...

    let buffer = inner.buffer.as_ref().unwrap();

    // NEW: Validate buffer against constraints
    if let Some(constraints) = inner.constraints.as_ref() {
        if let Err(reason) = validate_buffer(buffer, constraints) {
            inner.fail(resource, reason);
            return;
        }
    } else {
        // No constraints means source was never valid
        inner.fail(resource, FailureReason::BufferConstraints);
        return;
    }

    // ... proceed to frame() callback ...
}
```

### 6.3 Size Validation Policy

**Recommendation:** Allow buffers >= constraint size (like COSMIC)

**Rationale:**
- Protocol says buffer must "match" dimensions, but doesn't specify exact
- Allowing larger buffers is more permissive and client-friendly
- wlroots' exact match is stricter than necessary
- Damage regions handle the actual capture area

### 6.4 DRM Node Validation

**Question:** Should we validate DRM node matches?

**Recommendation:** No, for now.

**Rationale:**
- Not all compositors set DRM node on dmabufs
- `Dmabuf::node()` is only a "hint" per docs
- Format + modifier validation is sufficient
- Can add later if needed

---

## 7. Implementation Plan

### 7.1 Phase 1: Core Validation (Recommended First)

1. Add validation helper functions in `mod.rs`
2. Integrate into `Capture` request handler
3. Add necessary imports

**Estimated changes:** ~60-80 lines

### 7.2 Phase 2: Error Messaging (Optional Enhancement)

Add `tracing::debug!` for validation failures to aid debugging:

```rust
debug!(
    buffer_size = ?size,
    constraint_size = ?constraints.size,
    "Buffer size validation failed"
);
```

### 7.3 Phase 3: Documentation

Update module docs to document:
- Validation behavior
- Supported buffer types
- Error conditions

### 7.4 Testing Considerations

- Unit tests for validation helpers would require mocking WlBuffer
- Integration testing via weston-screenshooter or similar clients
- Verify graceful failure doesn't crash client

---

## 8. Appendix: Protocol Error vs Failure

**Protocol Errors** (kill the client):
- `already_captured` - capture sent twice
- `no_buffer` - capture without attach_buffer
- `invalid_buffer_damage` - negative coordinates

**Failures** (graceful, client can retry):
- `buffer_constraints` - buffer doesn't match
- `stopped` - session ended
- `unknown` - runtime error

Buffer validation failures should use `FailureReason::BufferConstraints`, NOT protocol errors.

---

## 9. Conclusion

Buffer validation is essential for a robust implementation. The recommended approach:

1. **Validate in library** before `frame()` callback
2. **Minimum size match** (>= constraints, not exact)
3. **Full format/modifier validation** for both SHM and DMA-BUF
4. **Use existing Smithay utilities** (`buffer_type`, `with_buffer_contents`, `get_dmabuf`)

This approach is consistent with COSMIC and other production compositors while providing automatic protection for all Smithay-based compositors.

---

*Document prepared for Smithay upstream contribution. Based on analysis of wlroots, COSMIC, Jay, and the ext-image-copy-capture-v1 protocol specification.*
