# Handover: ext-image-capture Protocol Implementation - Session 2

**Date:** 2026-01-10
**Status:** Awaiting maintainer response on design and licensing
**Owner:** Greg Lamberson, Lamco Development

---

## Quick Context

You are implementing `ext-image-capture-source-v1` and `ext-image-copy-capture-v1` protocols for Smithay. A comment was just posted to the GitHub issue proposing a design. Waiting for maintainer responses.

**GitHub Issue:** https://github.com/Smithay/smithay/issues/1900

---

## What Just Happened

1. Analyzed ids1024's requirements for an **extensible API** that allows downstream compositors (like COSMIC) to define custom capture source types (e.g., workspace capture)

2. Studied Smithay patterns across multiple protocol implementations

3. Evaluated 4 design options:
   - Option A: Associated type on handler (like SeatHandler)
   - Option B: Trait object with downcasting
   - Option C: Enum + UserDataMap hybrid
   - **Option D: Callback-based factory (SELECTED)**

4. Wrote comprehensive analysis document

5. Posted comment to GitHub issue proposing Option D design

---

## Selected Design: Option D (Callback-Based)

```rust
pub trait ImageCaptureSourceHandler {
    fn image_capture_source_state(&mut self) -> &mut ImageCaptureSourceState;

    fn output_source_created(&mut self, source: ImageCaptureSource, output: &Output) {
        source.user_data().insert_if_missing(|| MyOutputData::from(output));
    }
}
```

**Why this approach:**
- No generic parameters infecting the API
- Compositors store their representation in `source.user_data()`
- COSMIC can add workspace capture via custom global + user_data
- Follows existing Smithay patterns (similar to DmabufHandler::dmabuf_imported)

---

## Key Files

| File | Purpose |
|------|---------|
| `ANALYSIS-EXT-IMAGE-CAPTURE-EXTENSIBILITY.md` | Full design analysis with 4 options, code examples, roadmap |
| `HANDOVER-EXT-IMAGE-CAPTURE.md` | Original handover (background, protocol specs, Smithay orientation) |
| `cosmic-comp-reference/screencopy.rs` | Reference: cosmic-comp's screencopy implementation (~900 lines) |
| `cosmic-comp-reference/image_capture_source.rs` | Reference: cosmic-comp's source implementation |
| `src/wayland/image_capture_source/mod.rs` | Existing scaffolding (~410 lines) - will need refactoring |

---

## Pending Actions

### Waiting For:
1. **ids1024** - Confirmation that callback-based design direction works
2. **ids1024 + @Drakulix** - Explicit re-licensing confirmation (GPL → MIT) for cosmic-comp code

### GitHub Comment Posted (by glamberson):
```
Thanks for the feedback, @ids1024. The extensibility requirement aligns with what I need as well.
Also I appreciate your swift response as I'm frankly on fire to get this moving.

I've looked at the cosmic-comp implementation and Smithay's existing patterns. Based on your
requirements, I'm proposing a callback-based design where:

1. ImageCaptureSource is an opaque handle with a UserDataMap for compositor-specific data
2. Handler callbacks notify the compositor when sources are created, allowing it to store its representation
3. A register_custom_source_manager() method allows compositors to add custom globals (like workspace capture) that create the same ImageCaptureSource type
4. The screencopy handler receives ImageCaptureSource and calls back to the compositor for constraints and capture

[code snippet]

This avoids generic parameters, gives compositors full control over source representation, and
allows COSMIC to keep its workspace capture extension.

I have an immediate need for these protocols and am ready to implement. Before I proceed:

1. Does this design direction work?
2. For the screencopy implementation, I'd like to adapt cosmic-comp's screencopy.rs as a foundation. Could you and @Drakulix confirm the re-licensing to MIT is acceptable? I'll include attribution.

Thanks.
```

---

## When Responses Come In

### If design approved:

**Phase 1: Refactor image_capture_source**
1. Replace `CaptureSourceType` enum with opaque `ImageCaptureSource` + `UserDataMap`
2. Add callback-based handler trait
3. Add `register_custom_source_manager()` for extensions
4. Ensure selective capability (output-only vs output+toplevel)

**Phase 2: Implement image_copy_capture (screencopy)**
1. Port cosmic-comp's `screencopy.rs`
2. Replace `ImageCaptureSourceData` with `ImageCaptureSource`
3. Convert to Smithay handler pattern
4. Include proper attribution for re-licensed code

**Phase 3: Anvil integration + testing**

### If design needs changes:
Iterate based on feedback, update analysis document

---

## Key Maintainers

- **@ids1024** (Ian Douglas Scott) - System76/COSMIC, wrote cosmic-comp implementation
- **@Drakulix** (Victoria Brekenfeld) - Co-author of cosmic-comp screencopy code
- **@YaLTeR** - Niri compositor, mentioned by ids1024 for potential input

---

## Commands

```bash
# Check for responses
gh issue view 1900 --repo Smithay/smithay --comments | tail -50

# Build smithay
cargo build -p smithay --features wayland_frontend

# Check workspace
cargo check --workspace --all-targets
```

---

## Licensing Note

When re-licensing is confirmed, include attribution:
```rust
// Based on cosmic-comp's screencopy implementation by Victoria Brekenfeld (@Drakulix)
// and Ian Douglas Scott (@ids1024) at System76, re-licensed from GPL-3.0 to MIT.
```

---

**Next action:** Check for GitHub responses, then proceed with implementation or iterate on design.
