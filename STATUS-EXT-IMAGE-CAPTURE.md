# ext-image-capture Protocol Implementation Status

**Date:** 2026-01-10
**Status:** DRAFT PR SUBMITTED - Awaiting maintainer review
**Issue:** https://github.com/Smithay/smithay/issues/1900
**PR:** https://github.com/Smithay/smithay/pull/1902

---

## Current State

Draft PR #1902 submitted to Smithay. Implementation complete and tested. Awaiting maintainer review.

---

## Implementation Summary

| Component | Status | Lines |
|-----------|--------|-------|
| `ext-image-capture-source-v1` | Complete | ~500 |
| `ext-image-copy-capture-v1` | Complete | ~1390 |
| Buffer validation | Complete | +70 |
| Anvil integration | Complete | +60 |

**Total new code:** ~2000 lines

---

## Files Modified/Created

```
src/wayland/image_capture_source/mod.rs  - Modified (extensible design)
src/wayland/image_copy_capture/mod.rs    - Created (full implementation)
src/wayland/mod.rs                       - Modified (module registration)
anvil/src/state.rs                       - Modified (handler implementations)
```

---

## Build Status

```
cargo build          - Clean
cargo build -p anvil - Clean
cargo clippy         - 1 pre-existing warning (variant size, unrelated)
cargo fmt --check    - Clean
```

---

## Design Decisions

### Extensibility (Option D - Callback-Based)

Selected approach uses callbacks + UserDataMap for compositor-specific data:

```rust
pub trait ImageCaptureSourceHandler {
    fn image_capture_source_state(&mut self) -> &mut ImageCaptureSourceState;

    fn output_source_created(&mut self, source: ImageCaptureSource, output: &Output) {
        source.user_data().insert_if_missing(|| output.downgrade());
    }
}
```

**Rationale:** No generic parameters, compositors store data in user_data(), follows existing Smithay patterns.

### Buffer Validation

Validates buffers at Capture request time (before frame callback):
- SHM: size >= constraints, format in allowed list
- DMA-BUF: size >= constraints, format+modifier in allowed list
- Uses minimum-size matching (like COSMIC) not exact-match (like wlroots)

**Analysis document:** `ANALYSIS-BUFFER-VALIDATION.md`

---

## Pending Actions

1. **Await maintainer review** - Draft PR #1902 open
2. **Address feedback** - Iterate based on review comments
3. **Mark ready for review** - When feedback addressed

---

## Key Contacts

| Person | Role | Context |
|--------|------|---------|
| @ids1024 | Maintainer | Pointed to cosmic-comp, active on issue |
| @Drakulix | Maintainer | Co-author of cosmic-comp screencopy |
| @YaLTeR | Niri maintainer | May have input on design |

---

## Related Documents

| Document | Location |
|----------|----------|
| Analysis (buffer validation) | `~/smithay/ANALYSIS-BUFFER-VALIDATION.md` |
| Upstream tracking | `~/lamco-admin/upstream/smithay/` |
| Issue details | `~/lamco-admin/upstream/smithay/issues/1900-image-capture-protocols.md` |
| cosmic-comp reference | `~/smithay/cosmic-comp-reference/` |

---

## To Resume Work

1. Check GitHub issue #1900 for new comments
2. If ready to submit PR:
   ```bash
   cd ~/smithay
   git status  # Review changes
   git diff    # Inspect modifications
   # Create branch, commit, push, open PR
   ```
3. PR should reference issue #1900 and include attribution to cosmic-comp

---

## Attribution

Code adapted from cosmic-comp with attribution:
```rust
// Based on cosmic-comp's screencopy implementation by Victoria Brekenfeld (@Drakulix)
// and Ian Douglas Scott (@ids1024) at System76.
// Original source: https://github.com/pop-os/cosmic-comp/blob/master/src/wayland/protocols/screencopy.rs
```

---

## Quick Verification Commands

```bash
# Build
cargo build && cargo build -p anvil

# Lint
cargo clippy && cargo fmt --check

# Check file sizes
wc -l src/wayland/image_capture_source/mod.rs src/wayland/image_copy_capture/mod.rs

# Git status
git status
git diff --stat
```

---

*Last updated: 2026-01-10*
