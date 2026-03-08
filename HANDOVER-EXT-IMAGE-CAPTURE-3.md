# Handover: ext-image-capture Protocol Implementation - Session 3

**Date:** 2026-01-10
**Status:** Implementation Complete - Ready for Review
**Owner:** Greg Lamberson, Lamco Development

---

## Quick Summary

Both `ext-image-capture-source-v1` and `ext-image-copy-capture-v1` protocols are now implemented in Smithay with Anvil integration. The implementation follows Option D (callback-based) design as discussed in the GitHub issue.

**GitHub Issue:** https://github.com/Smithay/smithay/issues/1900

---

## What Was Accomplished

### Phase 1: Refactored image_capture_source (COMPLETE)

**File:** `src/wayland/image_capture_source/mod.rs`

Changes made:
1. Removed closed `CaptureSourceType` enum
2. Added `UserDataMap` to `ImageCaptureSource` for extensibility
3. Added handler callbacks for compositor notification:
   - `output_source_created()` - called when output capture source created
   - `toplevel_source_created()` - called when toplevel capture source created
   - `source_destroyed()` - called when any source is destroyed
4. Added unique ID generation for sources
5. Added `create_source()` method for custom protocols (workspace capture, etc.)
6. Added `from_resource()` for lookup from protocol objects

### Phase 2: Implemented image_copy_capture (COMPLETE)

**File:** `src/wayland/image_copy_capture/mod.rs` (~1320 lines, new file)

Implementation includes:
1. **BufferConstraints** - size, SHM formats, optional DMA-BUF constraints
2. **Session/SessionRef** - RAII pattern for capture sessions
   - `SessionRef` is cloneable for passing around
   - `Session` is owned wrapper that sends `stopped` on drop
3. **CursorSession/CursorSessionRef** - Same pattern for cursor capture
4. **Frame** - RAII frame that auto-fails if not completed
   - `frame.success()` signals completion with transform, damage, timestamp
   - `frame.fail()` signals failure with reason
   - Drop without calling either sends `Unknown` failure
5. **ImageCopyCaptureHandler** trait with callbacks:
   - `capture_constraints()` - return constraints for a source
   - `new_session()` - compositor stores the session
   - `frame()` - compositor performs capture
   - Optional cursor methods
6. **delegate_image_copy_capture!** macro

Attribution included:
```rust
// Based on cosmic-comp's screencopy implementation by Victoria Brekenfeld (@Drakulix)
// and Ian Douglas Scott (@ids1024) at System76.
// Original source: https://github.com/pop-os/cosmic-comp/blob/master/src/wayland/protocols/screencopy.rs
```

### Phase 3: Anvil Integration (COMPLETE)

**File:** `anvil/src/state.rs`

Changes made:
1. Added imports for both protocol modules
2. Added state fields to `AnvilState`:
   - `image_capture_source_state: ImageCaptureSourceState`
   - `image_copy_capture_state: ImageCopyCaptureState`
3. Implemented `ImageCaptureSourceHandler`:
   - Stores `WeakOutput` in source's user_data for later lookup
4. Implemented `ImageCopyCaptureHandler`:
   - Returns basic SHM constraints (Argb8888, Xrgb8888)
   - Fails all frames with `Unknown` (actual capture not implemented in Anvil)
5. Added delegate macros
6. Initialize states in `init()`

---

## Build Status

```bash
# Full workspace build (excluding wlcs_anvil which has pre-existing issues)
cargo build --workspace --exclude wlcs_anvil --all-targets  # SUCCESS

# Clippy
cargo clippy --workspace --exclude wlcs_anvil --all-targets  # CLEAN

# Format
cargo fmt --check  # CLEAN
```

Note: `wlcs_anvil` has pre-existing API compatibility issues with `wayland_sys` (unrelated to these changes).

---

## Key Design Decisions

### 1. Extensibility via UserDataMap
Compositors store their representation of capture sources in `source.user_data()`. This allows:
- COSMIC to add workspace capture via custom global
- No generic parameters infecting the API
- Clean separation between protocol handling and compositor logic

### 2. RAII Lifecycle Management
- `Session` and `CursorSession` are owned wrappers that cleanup on drop
- `Frame` auto-fails if not explicitly completed
- This prevents resource leaks and ensures protocol correctness

### 3. Smithay Pattern Compliance
- Handler trait with `*_state()` accessor
- `delegate_*!` macro for dispatch delegation
- GlobalDispatch with client filter support
- Arc<Mutex<Inner>> + UserDataMap pattern for thread-safe sharing

---

## Files Modified/Created

| File | Status | Lines |
|------|--------|-------|
| `src/wayland/image_capture_source/mod.rs` | Modified | ~500 |
| `src/wayland/image_copy_capture/mod.rs` | **Created** | ~1320 |
| `src/wayland/mod.rs` | Modified | +1 |
| `anvil/src/state.rs` | Modified | +60 |

---

## What's NOT Implemented

1. **Actual capture in Anvil** - Frames are failed with `Unknown`. Real compositors need to:
   - Render the source to the provided buffer
   - Call `frame.success()` with transform, damage, and presentation timestamp

2. **DMA-BUF support in Anvil** - Constraints only include SHM formats. DRM-backed compositors should:
   - Populate `DmabufConstraints` with render node and formats
   - Handle DMA-BUF import in frame capture

3. **Cursor capture in Anvil** - `cursor_capture_constraints()` returns `None` (not supported)

---

## Testing Recommendations

1. **Protocol conformance**: Use `wayland-protocols` test clients or weston's simple-screencopy
2. **Session lifecycle**: Verify `stopped` event on session drop
3. **Frame lifecycle**: Verify auto-fail on frame drop without success/fail
4. **Custom sources**: Test that compositors can create custom source types

---

## Next Steps

1. **GitHub PR** - Submit for review
2. **Maintainer feedback** - Address any design concerns
3. **Documentation** - Add module-level docs if requested
4. **Real capture** - Interested compositors can implement actual rendering

---

## Reference Files

| File | Purpose |
|------|---------|
| `ANALYSIS-EXT-IMAGE-CAPTURE-EXTENSIBILITY.md` | Design analysis with 4 options evaluated |
| `HANDOVER-EXT-IMAGE-CAPTURE.md` | Original context and protocol specs |
| `HANDOVER-EXT-IMAGE-CAPTURE-2.md` | Design selection and GitHub discussion |
| `cosmic-comp-reference/screencopy.rs` | Original cosmic-comp implementation |
| `cosmic-comp-reference/image_capture_source.rs` | Original cosmic-comp source impl |

---

## Commands

```bash
# Build everything
cargo build --workspace --exclude wlcs_anvil --all-targets

# Check with clippy
cargo clippy --workspace --exclude wlcs_anvil --all-targets

# Format check
cargo fmt --check

# View implementation
less src/wayland/image_copy_capture/mod.rs
less src/wayland/image_capture_source/mod.rs
less anvil/src/state.rs
```

---

## Key Maintainers

- **@ids1024** (Ian Douglas Scott) - System76/COSMIC, original cosmic-comp author
- **@Drakulix** (Victoria Brekenfeld) - Co-author of cosmic-comp screencopy
- **@YaLTeR** - Niri compositor maintainer

---

**Status:** Ready for PR submission and review.
