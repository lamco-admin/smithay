# Handover: ext-image-capture Protocol Implementation for Smithay

**Date:** 2026-01-08
**Status:** Ready to begin implementation
**Owner:** Greg Lamberson, Lamco Development

---

## Quick Start

You are implementing `ext-image-capture-source-v1` and `ext-image-copy-capture-v1` protocols for Smithay. This is an upstream contribution that will also serve Lamco's Smithay-based compositor.

**GitHub Issue:** https://github.com/Smithay/smithay/issues/1900 (opened, awaiting feedback)

**Do not wait for feedback to begin.** Start with Phase 1 (ext-image-capture-source-v1) which is non-controversial. Adjust based on maintainer input as it arrives.

---

## Context

### Why This Work

1. **Lamco's compositor** (at `~/wayland/wrd-server-specs/`, branch `feature/lamco-compositor-clipboard`) needs screen capture for client plugins (WASM/WebGPU clients)
2. **Smithay lacks these protocols** - blank entries in issue #781
3. **Contributing upstream** benefits both Lamco and the ecosystem (Niri, Cosmic, MagmaWM, etc.)

### What the Protocols Do

```
ext-image-capture-source-v1:
  Creates opaque "source" handles from outputs or toplevels
  ↓
ext-image-copy-capture-v1:
  Uses sources to create capture sessions
  Clients attach buffers, request capture, receive pixels
```

Clients (like grim, OBS, or Lamco's WASM plugins) use these to capture screen content.

---

## Key Documents

| Document | Location | Purpose |
|----------|----------|---------|
| **Comprehensive Analysis** | `~/wayland/wrd-server-specs/docs/research/EXT-IMAGE-CAPTURE-PROTOCOLS-ANALYSIS.md` | Full protocol spec, implementation guide, code patterns, roadmap |
| **Issue Tracking** | `~/lamco-admin/upstream/smithay/issues/1900-image-capture-protocols.md` | Activity log for the GitHub issue |
| **Upstream Project** | `~/lamco-admin/upstream/smithay/README.md` | Overall Smithay contribution tracking |

---

## Smithay Codebase Orientation

### Directory Structure

```
~/smithay/
├── src/
│   ├── wayland/           # Protocol implementations (YOUR TARGET)
│   │   ├── dmabuf/        # Reference: buffer handling patterns
│   │   ├── drm_syncobj/   # Reference: clean modern pattern
│   │   ├── foreign_toplevel_list/  # Reference: handle pattern
│   │   └── ...
│   └── backend/
│       └── renderer/
│           └── mod.rs     # ExportMem trait (frame readback)
├── anvil/                 # Example compositor (integration target)
│   └── src/
│       └── state.rs       # Where protocols are integrated
└── Cargo.toml
```

### Protocol Implementation Pattern

Every Smithay protocol follows this structure:

```rust
// 1. State struct
pub struct ProtocolState {
    global: GlobalId,
    // ... additional state
}

// 2. Handler trait
pub trait ProtocolHandler: /* bounds */ {
    fn protocol_state(&mut self) -> &mut ProtocolState;
    // Optional callbacks with defaults
}

// 3. GlobalDispatch + Dispatch implementations
impl<D: ProtocolHandler> GlobalDispatch<...> for ProtocolState { ... }
impl<D: ProtocolHandler> Dispatch<...> for ProtocolState { ... }

// 4. Delegate macro
#[macro_export]
macro_rules! delegate_protocol {
    ($ty:ty) => { /* generates dispatch delegations */ };
}
```

### Key Files to Study

| File | Lines | Why Study It |
|------|-------|--------------|
| `src/wayland/drm_syncobj/mod.rs` | ~540 | Clean, modern pattern |
| `src/wayland/foreign_toplevel_list/mod.rs` | ~540 | Handle pattern (Arc<Mutex<Inner>>) |
| `src/wayland/dmabuf/mod.rs` | ~1200 | Buffer handling, complex reference |
| `src/wayland/pointer_warp.rs` | ~210 | Simplest complete example |
| `src/backend/renderer/mod.rs` | ~800 | ExportMem trait (lines 720-777) |
| `anvil/src/state.rs` | ~1150 | Protocol integration example |

---

## Implementation Plan

### Phase 1: ext-image-capture-source-v1 (~16h)

**Goal:** Create source handles from outputs and toplevels

**Files to create:**
```
src/wayland/image_capture_source/
└── mod.rs
```

**Core types:**
- `ImageCaptureSourceState` - holds globals
- `ImageCaptureSource` - handle to a capture source (Arc<Mutex<Inner>>)
- `CaptureSourceType` - enum: Output(Output) | Toplevel(ForeignToplevelHandle)
- `ImageCaptureSourceHandler` - trait for compositor implementation
- `delegate_image_capture_source!` - macro

**Start here.** This is simpler and validates your understanding of Smithay patterns.

### Phase 2: ext-image-copy-capture-v1 (~48h)

**Goal:** Session management, frame capture, buffer handling

**Files to create:**
```
src/wayland/image_copy_capture/
├── mod.rs
└── dispatch.rs (optional, for complex request handling)
```

**Core types:**
- `ImageCopyCaptureState` - holds global
- `CaptureSession` - active capture session
- `CaptureFrameState` - per-frame state
- `CaptureBufferConstraints` - format/size requirements
- `ImageCopyCaptureHandler` - trait with `capture_frame()` method
- `delegate_image_copy_capture!` - macro

**Key integration:** Use `ExportMem::copy_framebuffer()` for actual pixel readback.

### Phase 3: Anvil Integration (~4h)

Add to `anvil/src/state.rs`:
- State fields for both protocols
- Handler trait implementations
- Delegate macro invocations

### Phase 4: Testing (~8h)

- Test with `grim` (if updated for new protocol) or write test client
- Verify buffer constraints are communicated correctly
- Test both SHM and DMA-BUF paths

---

## Protocol Specifications Summary

### ext-image-capture-source-v1

```
Interfaces:
  ext_output_image_capture_source_manager_v1
    └─ create_source(wl_output) → ext_image_capture_source_v1

  ext_foreign_toplevel_image_capture_source_manager_v1
    └─ create_source(toplevel_handle) → ext_image_capture_source_v1

  ext_image_capture_source_v1
    └─ (opaque handle, no requests/events)
```

### ext-image-copy-capture-v1

```
Interfaces:
  ext_image_copy_capture_manager_v1
    ├─ create_session(source, options) → session
    └─ create_pointer_cursor_session(source, device) → cursor_session

  ext_image_copy_capture_session_v1
    Events: buffer_size, shm_format, dmabuf_device, dmabuf_format, done, stopped
    Requests: create_frame, destroy

  ext_image_copy_capture_frame_v1
    Requests: attach_buffer, damage_buffer, capture, destroy
    Events: transform, damage, presentation_time, ready, failed

  ext_image_copy_capture_cursor_session_v1
    Events: enter, leave, position, hotspot
```

### Capture Flow

1. Client binds manager, creates source from output
2. Client creates session from source
3. Compositor sends buffer constraints (size, formats), then `done`
4. Client allocates matching buffer
5. Client creates frame, attaches buffer, calls `capture`
6. Compositor renders to buffer using `ExportMem::copy_framebuffer()`
7. Compositor sends metadata events, then `ready`
8. Client reads buffer

---

## Smithay Contribution Guidelines

### Style

- No rustfmt.toml, use default `cargo fmt`
- Follow existing patterns exactly
- Documentation with "How to use" examples in module docs
- Feature flags for optional protocols

### Process

1. Issue opened (#1900) - done
2. Implement following patterns from drm_syncobj/foreign_toplevel_list
3. Open draft PR early for feedback
4. Iterate based on review
5. Include anvil integration

### Communication

- Matrix: #smithay:matrix.org (primary)
- IRC: #smithay on libera.chat (bridged)
- Maintainers: @Drakulix, @PolyMeilex, @ids1024

---

## ExportMem Trait Reference

This is how you'll actually capture frames:

```rust
// From src/backend/renderer/mod.rs lines 720-777

pub trait ExportMem: Renderer {
    type TextureMapping: TextureMapping;

    /// Copy framebuffer contents to a mapping
    fn copy_framebuffer(
        &mut self,
        target: &Self::Framebuffer<'_>,
        region: Rectangle<i32, BufferCoord>,
        format: Fourcc,
    ) -> Result<Self::TextureMapping, Self::Error>;

    /// Copy texture contents to a mapping
    fn copy_texture(
        &mut self,
        texture: &Self::TextureId,
        region: Rectangle<i32, BufferCoord>,
        format: Fourcc,
    ) -> Result<Self::TextureMapping, Self::Error>;

    /// Check if texture can be read
    fn can_read_texture(&mut self, texture: &Self::TextureId) -> Result<bool, Self::Error>;

    /// Get raw pointer to mapping data
    fn map_texture<'a>(&mut self, mapping: &'a Self::TextureMapping) -> Result<&'a [u8], Self::Error>;
}
```

Implemented by: GlesRenderer, PixmanRenderer, GlowRenderer, MultiRenderer

---

## Immediate Next Steps

1. **Read** `src/wayland/drm_syncobj/mod.rs` thoroughly (~30 min)
2. **Read** `src/wayland/foreign_toplevel_list/mod.rs` for handle pattern (~30 min)
3. **Create** `src/wayland/image_capture_source/mod.rs` skeleton
4. **Implement** `ImageCaptureSourceState` and globals
5. **Implement** `ImageCaptureSource` handle type
6. **Implement** handler trait and dispatch
7. **Add** delegate macro
8. **Test** compilation: `cargo build -p smithay --features wayland_frontend`

---

## Commands

```bash
# Build smithay with wayland features
cargo build -p smithay --features wayland_frontend

# Check everything
cargo check --workspace --all-targets

# Run clippy
cargo clippy --workspace --all-targets -- -D warnings

# Format
cargo fmt --all

# Build anvil (for integration testing)
cargo build -p anvil
```

---

## Questions to Resolve

These may come up during implementation:

1. **Permission model:** Should there be client filtering? (Likely yes, follow dmabuf pattern)
2. **Toplevel dependency:** Require ForeignToplevelListState for toplevel capture?
3. **Cursor session priority:** Implement in Phase 2 or defer?

Check GitHub issue #1900 for maintainer responses before making decisions.

---

## Files Modified/Created Checklist

```
[ ] src/wayland/mod.rs                    # Add module exports
[ ] src/wayland/image_capture_source/mod.rs   # New file
[ ] src/wayland/image_copy_capture/mod.rs     # New file
[ ] src/wayland/image_copy_capture/dispatch.rs # New file (optional)
[ ] anvil/src/state.rs                    # Integration
[ ] Cargo.toml                            # Feature flags if needed
```

---

**Ready to begin. Start with Phase 1.**
