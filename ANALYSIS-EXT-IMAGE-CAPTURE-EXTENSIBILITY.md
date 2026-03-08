# Comprehensive Analysis: Extensible Image Capture Protocol Design for Smithay

**Date:** 2026-01-10
**Author:** Greg Lamberson, Lamco Development
**Status:** Design Analysis for Maintainer Review
**Related Issue:** https://github.com/Smithay/smithay/issues/1900

---

## Executive Summary

This document analyzes the design requirements and implementation options for adding `ext-image-capture-source-v1` and `ext-image-copy-capture-v1` protocols to Smithay. The analysis is driven by maintainer feedback from ids1024 (Ian Douglas Scott, System76/COSMIC) requesting an **extensible API** that allows downstream compositors to define custom capture source types.

**Key Requirement:** The Smithay implementation must allow compositors like cosmic-comp to extend the capture source abstraction with custom types (e.g., workspace capture) without modifying Smithay itself.

---

## Table of Contents

1. [Background and Context](#1-background-and-context)
2. [Maintainer Requirements Analysis](#2-maintainer-requirements-analysis)
3. [Smithay Pattern Analysis](#3-smithay-pattern-analysis)
4. [cosmic-comp Reference Implementation Analysis](#4-cosmic-comp-reference-implementation-analysis)
5. [Extensibility Design Options](#5-extensibility-design-options)
6. [Recommended Approach](#6-recommended-approach)
7. [Implementation Roadmap](#7-implementation-roadmap)
8. [Licensing and Attribution](#8-licensing-and-attribution)
9. [Suggested Communication](#9-suggested-communication)

---

## 1. Background and Context

### 1.1 The Protocols

**ext-image-capture-source-v1** creates opaque "source" handles representing capturable resources:
```
ext_output_image_capture_source_manager_v1
  └─ create_source(wl_output) → ext_image_capture_source_v1

ext_foreign_toplevel_image_capture_source_manager_v1
  └─ create_source(toplevel_handle) → ext_image_capture_source_v1
```

**ext-image-copy-capture-v1** uses sources to perform actual screen capture:
```
ext_image_copy_capture_manager_v1
  ├─ create_session(source, options) → session
  └─ create_pointer_cursor_session(source, device) → cursor_session
```

### 1.2 Current State

- **Smithay:** No implementation (blank entries in issue #781)
- **cosmic-comp:** Full implementation under GPL-3.0 (relicensable to MIT per maintainers)
- **Niri:** Uses older wlr-screencopy, not the new ext protocols
- **Your scaffolding:** Basic `image_capture_source` implementation at `src/wayland/image_capture_source/mod.rs`

### 1.3 Why Extensibility Matters

COSMIC needs to capture **workspaces** (virtual desktops) in addition to outputs and toplevels. This requires a custom protocol extension (`zcosmic_workspace_image_capture_source_manager_v1`) that creates the same `ext_image_capture_source_v1` object type but from COSMIC-specific workspace handles.

Without an extensible design, COSMIC would need to:
1. Fork Smithay's implementation, or
2. Maintain parallel screencopy code, or
3. Convince Smithay to add COSMIC-specific code upstream (inappropriate)

---

## 2. Maintainer Requirements Analysis

### 2.1 ids1024's Explicit Requirements

From [issue #1900 comment](https://github.com/Smithay/smithay/issues/1900):

> "I think ideally we want an API that's extensible. So a compositor like cosmic-comp could use the smithay abstraction, but define an additional type of capture source with its own protocol."

> "A compositor may also want to implement one capture source type, but not another. So there should also probably be a way to expose only the output image capture source but not the toplevel one, or vice versa."

### 2.2 Derived Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| R1 | Custom capture source types can be defined by downstream compositors | **Critical** |
| R2 | Built-in support for Output and Toplevel capture sources | High |
| R3 | Compositors can expose only a subset of source types | High |
| R4 | Clean integration with screencopy (image-copy-capture) protocol | High |
| R5 | No COSMIC-specific code in Smithay | Critical |
| R6 | Follow Smithay patterns and style | High |

### 2.3 Licensing Confirmation

ids1024 confirmed:
> "cosmic-comp is GPL licensed, while Smithay is MIT license. But the image-copy code there is all written by @Drakulix and me at System76, and I don't think there should be any problem with re-licensing that part for inclusion in Smithay"

This confirms we can use cosmic-comp's implementation as a foundation.

---

## 3. Smithay Pattern Analysis

### 3.1 Handler Trait Pattern (Standard)

Every Smithay protocol follows this structure:

```rust
pub trait ProtocolHandler:
    GlobalDispatch<ProtocolManager, GlobalData>
    + Dispatch<ProtocolManager, ()>
    + Dispatch<ProtocolObject, ObjectData>
    + 'static
{
    fn protocol_state(&mut self) -> &mut ProtocolState;

    // Optional callbacks with defaults
    fn on_event(&mut self, ...) {}
}
```

**Examples:** `DrmSyncobjHandler`, `ForeignToplevelListHandler`, `DmabufHandler`

### 3.2 Associated Type Pattern (Extensibility)

Used when compositors need to define custom types:

```rust
pub trait SeatHandler: Sized {
    type KeyboardFocus: KeyboardTarget<Self> + PartialEq + Clone + 'static;
    type PointerFocus: PointerTarget<Self> + PartialEq + Clone + 'static;
    type TouchFocus: TouchTarget<Self> + PartialEq + Clone + 'static;

    fn seat_state(&mut self) -> &mut SeatState<Self>;
    fn focus_changed(&mut self, seat: &Seat<Self>, focused: Option<&Self::KeyboardFocus>) {}
}
```

**Key Insight:** This pattern lets compositors define their own `KeyboardFocus` type (e.g., cosmic-comp uses `CosmicSurface`), while Smithay handles the protocol mechanics.

### 3.3 UserDataMap Pattern (Runtime Extension)

Used for attaching compositor-specific data to Smithay objects:

```rust
pub struct ForeignToplevelHandle {
    inner: Arc<(Mutex<ForeignToplevelHandleInner>, UserDataMap)>,
}

impl ForeignToplevelHandle {
    pub fn user_data(&self) -> &UserDataMap {
        &self.inner.1
    }
}
```

Compositors can store any `Send + Sync + 'static` data.

### 3.4 Callback Pattern (Event Notification)

```rust
pub trait DmabufHandler: BufferHandler {
    fn dmabuf_state(&mut self) -> &mut DmabufState;

    fn dmabuf_imported(&mut self, global: &DmabufGlobal, dmabuf: Dmabuf, notifier: ImportNotifier);

    fn new_surface_feedback(
        &mut self,
        surface: &WlSurface,
        global: &DmabufGlobal,
    ) -> Option<DmabufFeedback> {
        None // default implementation
    }
}
```

### 3.5 Selective Global Registration Pattern

Multiple `new_*` constructors for different capability levels:

```rust
impl ImageCaptureSourceState {
    pub fn new<D>(display: &DisplayHandle) -> Self { ... }  // output only
    pub fn new_with_toplevel_capture<D>(display: &DisplayHandle) -> Self { ... }  // output + toplevel
}
```

---

## 4. cosmic-comp Reference Implementation Analysis

### 4.1 File Structure

```
cosmic-comp/src/wayland/protocols/
├── image_capture_source.rs  (~280 lines) - Source creation
└── screencopy.rs            (~900 lines) - Session/frame management
```

### 4.2 image_capture_source.rs Analysis

**Current Design (Closed Enum):**
```rust
#[derive(Debug, Clone, PartialEq)]
pub enum ImageCaptureSourceData {
    Output(WeakOutput),
    Workspace(WorkspaceHandle),    // COSMIC-specific
    Toplevel(CosmicSurface),       // COSMIC-specific type
    Destroyed,
}
```

**Issues:**
1. Uses `CosmicSurface` instead of Smithay's types
2. Hardcodes `Workspace` variant (COSMIC extension)
3. Not extensible by other compositors

**What Works Well:**
- Clean separation of manager types with separate global data
- Filter pattern for client visibility
- Simple dispatch implementation

### 4.3 screencopy.rs Analysis

**Strengths:**
- Complete ext-image-copy-capture-v1 implementation
- Robust RAII lifecycle management (`Session`, `CursorSession`, `Frame`)
- Proper buffer constraint validation (SHM + DMA-BUF)
- `UserDataMap` on sessions for compositor data

**Key Handler Trait:**
```rust
pub trait ScreencopyHandler {
    fn screencopy_state(&mut self) -> &mut ScreencopyState;

    fn capture_source(&mut self, source: &ImageCaptureSourceData) -> Option<BufferConstraints>;
    fn capture_cursor_source(&mut self, source: &ImageCaptureSourceData) -> Option<BufferConstraints>;

    fn new_session(&mut self, session: Session);
    fn new_cursor_session(&mut self, session: CursorSession);

    fn frame(&mut self, session: SessionRef, frame: Frame);
    fn cursor_frame(&mut self, session: CursorSessionRef, frame: Frame);

    fn frame_aborted(&mut self, frame_handle: FrameRef);
    fn session_destroyed(&mut self, session: SessionRef) { let _ = session; }
    fn cursor_session_destroyed(&mut self, session: CursorSessionRef) { let _ = session; }
}
```

**Dependency on Source Data:**
The screencopy handler receives `ImageCaptureSourceData` and must understand it to compute `BufferConstraints`. This is the critical extensibility point.

---

## 5. Extensibility Design Options

### 5.1 Option A: Associated Type on Handler

**Design:**
```rust
pub trait ImageCaptureSourceHandler {
    /// Compositor-defined capture source type
    type CaptureSource: Clone + Debug + Send + Sync + 'static;

    fn image_capture_source_state(&mut self) -> &mut ImageCaptureSourceState;

    /// Create source from output (required)
    fn create_output_source(&mut self, output: &Output) -> Option<Self::CaptureSource>;

    /// Create source from toplevel (optional, return None to disable)
    fn create_toplevel_source(&mut self, handle: &ForeignToplevelHandle) -> Option<Self::CaptureSource>;
}

// Screencopy handler uses the same associated type
pub trait ImageCopyCaptureHandler: ImageCaptureSourceHandler {
    fn image_copy_capture_state(&mut self) -> &mut ImageCopyCaptureState;

    fn capture_constraints(&mut self, source: &Self::CaptureSource) -> Option<BufferConstraints>;
    fn capture_frame(&mut self, session: &Session<Self::CaptureSource>, frame: Frame);
}
```

**COSMIC Usage:**
```rust
impl ImageCaptureSourceHandler for CosmicState {
    type CaptureSource = CaptureSourceType;  // COSMIC's enum with Workspace variant

    fn create_output_source(&mut self, output: &Output) -> Option<Self::CaptureSource> {
        Some(CaptureSourceType::Output(output.downgrade()))
    }

    fn create_toplevel_source(&mut self, handle: &ForeignToplevelHandle) -> Option<Self::CaptureSource> {
        // Map to CosmicSurface
        Some(CaptureSourceType::Toplevel(self.find_cosmic_surface(handle)?))
    }
}

// COSMIC can also register custom workspace manager global
// and implement its own Dispatch that creates CaptureSourceType::Workspace
```

**Pros:**
- Type-safe at compile time
- Zero runtime cost
- Follows Smithay's `SeatHandler` pattern
- Full control over source representation

**Cons:**
- Associated type "infects" generic bounds throughout
- More complex for simple compositors
- State structs become generic: `ImageCaptureSourceState<D::CaptureSource>`

### 5.2 Option B: Trait Object with Downcasting

**Design:**
```rust
pub trait CaptureSource: Debug + Send + Sync + 'static {
    fn as_any(&self) -> &dyn std::any::Any;
    fn clone_box(&self) -> Box<dyn CaptureSource>;
}

// Built-in implementations
impl CaptureSource for Output { ... }
impl CaptureSource for ForeignToplevelHandle { ... }

pub struct ImageCaptureSource {
    inner: Arc<dyn CaptureSource>,
}

impl ImageCaptureSource {
    pub fn downcast_ref<T: CaptureSource + 'static>(&self) -> Option<&T> {
        self.inner.as_any().downcast_ref()
    }
}
```

**COSMIC Usage:**
```rust
// COSMIC defines workspace source
impl CaptureSource for WorkspaceHandle { ... }

// In capture handler
fn capture_constraints(&mut self, source: &ImageCaptureSource) -> Option<BufferConstraints> {
    if let Some(output) = source.downcast_ref::<Output>() {
        return self.output_constraints(output);
    }
    if let Some(workspace) = source.downcast_ref::<WorkspaceHandle>() {
        return self.workspace_constraints(workspace);  // COSMIC extension
    }
    if let Some(toplevel) = source.downcast_ref::<ForeignToplevelHandle>() {
        return self.toplevel_constraints(toplevel);
    }
    None
}
```

**Pros:**
- Simple API, no generic parameters
- Easy for basic compositors
- Runtime flexibility

**Cons:**
- Runtime type checking (downcasting)
- Not exhaustive - can miss cases
- Object cloning overhead

### 5.3 Option C: Enum + UserDataMap Hybrid

**Design:**
```rust
/// Built-in source types
pub enum BuiltinCaptureSource {
    Output(WeakOutput),
    Toplevel(ForeignToplevelHandle),
}

pub struct ImageCaptureSource {
    builtin: Option<BuiltinCaptureSource>,
    user_data: UserDataMap,
}

impl ImageCaptureSource {
    pub fn builtin(&self) -> Option<&BuiltinCaptureSource> {
        self.builtin.as_ref()
    }

    pub fn user_data(&self) -> &UserDataMap {
        &self.user_data
    }
}
```

**COSMIC Usage:**
```rust
// For custom workspace source, builtin is None, data in user_data
source.user_data().insert_if_missing(|| WorkspaceHandle::from(workspace));

// In handler
fn capture_constraints(&mut self, source: &ImageCaptureSource) -> Option<BufferConstraints> {
    if let Some(builtin) = source.builtin() {
        match builtin {
            BuiltinCaptureSource::Output(o) => self.output_constraints(o),
            BuiltinCaptureSource::Toplevel(t) => self.toplevel_constraints(t),
        }
    } else if let Some(workspace) = source.user_data().get::<WorkspaceHandle>() {
        self.workspace_constraints(workspace)  // COSMIC extension
    } else {
        None
    }
}
```

**Pros:**
- Backwards compatible
- Simple for common cases
- No generic complexity

**Cons:**
- Two paths to check (builtin vs user_data)
- Awkward pattern matching
- Less type-safe

### 5.4 Option D: Callback-Based Factory

**Design:**
```rust
pub trait ImageCaptureSourceHandler {
    fn image_capture_source_state(&mut self) -> &mut ImageCaptureSourceState;

    /// Called when a source is created from an output
    fn output_source_created(
        &mut self,
        source: &ImageCaptureSource,
        output: &Output
    );

    /// Called when a source is created from a toplevel
    fn toplevel_source_created(
        &mut self,
        source: &ImageCaptureSource,
        handle: &ForeignToplevelHandle
    );
}

pub trait ImageCopyCaptureHandler {
    /// Compositor provides constraints for a source
    fn capture_constraints(&mut self, source: &ImageCaptureSource) -> Option<BufferConstraints>;

    /// Compositor performs the actual capture
    fn capture_frame(&mut self, session: &Session, frame: Frame);
}
```

**COSMIC Usage:**
```rust
impl ImageCaptureSourceHandler for CosmicState {
    fn output_source_created(&mut self, source: &ImageCaptureSource, output: &Output) {
        // Store mapping: source → output
        self.source_map.insert(source.id(), CaptureTarget::Output(output.clone()));
    }

    fn toplevel_source_created(&mut self, source: &ImageCaptureSource, handle: &ForeignToplevelHandle) {
        // Map to COSMIC surface
        if let Some(surface) = self.find_cosmic_surface(handle) {
            self.source_map.insert(source.id(), CaptureTarget::Toplevel(surface));
        }
    }
}

// For workspace extension, COSMIC handles its own protocol dispatch
// and updates source_map with CaptureTarget::Workspace
```

**Pros:**
- Full control to compositor
- Clean separation of concerns
- No generic complexity
- Easy to add new source types downstream

**Cons:**
- Compositor must maintain source→data mapping
- More boilerplate
- Slightly less efficient (extra lookup)

---

## 6. Recommended Approach

### 6.1 Primary Recommendation: Option D (Callback-Based Factory)

**Rationale:**

1. **Aligns with Smithay Philosophy:** "use what you want" library - compositors have full control
2. **Minimal Complexity:** No generic parameters infecting the API
3. **Maximum Flexibility:** Any source type, any mapping strategy
4. **Follows Existing Patterns:** Similar to `DmabufHandler::dmabuf_imported` and `ForeignToplevelListHandler`
5. **Clean Extension Path:** Compositors handle custom protocols entirely themselves, just update their internal mapping

### 6.2 Design Sketch

```rust
//! ext-image-capture-source-v1 protocol implementation
//!
//! ## Basic Usage (Output-only)
//!
//! ```no_run
//! use smithay::wayland::image_capture_source::{
//!     ImageCaptureSourceState, ImageCaptureSourceHandler, ImageCaptureSource,
//! };
//!
//! impl ImageCaptureSourceHandler for State {
//!     fn image_capture_source_state(&mut self) -> &mut ImageCaptureSourceState {
//!         &mut self.image_capture_source
//!     }
//!
//!     fn output_source_created(&mut self, source: ImageCaptureSource, output: &Output) {
//!         // Store for later use in capture
//!         source.user_data().insert_if_missing(|| output.downgrade());
//!     }
//! }
//!
//! // Create state with output capture only
//! let state = ImageCaptureSourceState::new::<State>(&display);
//! ```
//!
//! ## Extended Usage (Custom Source Types)
//!
//! ```no_run
//! // COSMIC can register additional globals for workspace capture
//! // and store WorkspaceHandle in source.user_data()
//! ```

use std::sync::{Arc, Mutex};
use wayland_server::{backend::GlobalId, /* ... */};

/// Opaque handle to a capture source.
///
/// Use `user_data()` to store compositor-specific data about what this source represents.
#[derive(Debug, Clone)]
pub struct ImageCaptureSource {
    inner: Arc<ImageCaptureSourceInner>,
}

#[derive(Debug)]
struct ImageCaptureSourceInner {
    id: usize,
    user_data: UserDataMap,
    alive: AtomicBool,
}

impl ImageCaptureSource {
    /// Unique identifier for this source (stable for lifetime of source)
    pub fn id(&self) -> usize {
        self.inner.id
    }

    /// Access compositor-specific data
    pub fn user_data(&self) -> &UserDataMap {
        &self.inner.user_data
    }

    /// Check if source is still valid
    pub fn alive(&self) -> bool {
        self.inner.alive.load(Ordering::Acquire)
    }

    /// Retrieve source from protocol resource
    pub fn from_resource(resource: &ExtImageCaptureSourceV1) -> Option<Self> {
        resource.data::<ImageCaptureSourceData>().map(|d| d.source.clone())
    }
}

/// Handler trait for image capture source protocol.
pub trait ImageCaptureSourceHandler:
    GlobalDispatch<ExtOutputImageCaptureSourceManagerV1, ImageCaptureSourceGlobalData>
    + Dispatch<ExtOutputImageCaptureSourceManagerV1, ()>
    + Dispatch<ExtImageCaptureSourceV1, ImageCaptureSourceData>
    + 'static
{
    /// State accessor
    fn image_capture_source_state(&mut self) -> &mut ImageCaptureSourceState;

    /// Called when a capture source is created from an output.
    ///
    /// Use `source.user_data()` to store your representation of this output for later capture.
    fn output_source_created(&mut self, source: ImageCaptureSource, output: &Output);

    /// Called when a capture source is created from a toplevel.
    ///
    /// Only called if toplevel capture is enabled via `new_with_toplevel_capture()`.
    fn toplevel_source_created(&mut self, source: ImageCaptureSource, handle: &ForeignToplevelHandle) {
        let _ = (source, handle);  // default: no-op
    }

    /// Called when a capture source is destroyed.
    fn source_destroyed(&mut self, source: ImageCaptureSource) {
        let _ = source;  // default: no-op
    }
}

/// State for image capture source protocol.
#[derive(Debug)]
pub struct ImageCaptureSourceState {
    output_manager_global: GlobalId,
    toplevel_manager_global: Option<GlobalId>,
    sources: Vec<WeakImageCaptureSource>,
}

impl ImageCaptureSourceState {
    /// Create with output capture only
    pub fn new<D: ImageCaptureSourceHandler>(display: &DisplayHandle) -> Self { ... }

    /// Create with output and toplevel capture
    pub fn new_with_toplevel_capture<D>(display: &DisplayHandle) -> Self
    where
        D: ImageCaptureSourceHandler
            + GlobalDispatch<ExtForeignToplevelImageCaptureSourceManagerV1, ImageCaptureSourceGlobalData>
            + Dispatch<ExtForeignToplevelImageCaptureSourceManagerV1, ()>
    { ... }

    /// Create with filter
    pub fn new_with_filter<D, F>(display: &DisplayHandle, filter: F) -> Self { ... }

    /// Register a custom source manager global (for extensions like workspace capture)
    ///
    /// Returns the GlobalId for the custom manager.
    pub fn register_custom_source_manager<D, M, G, F>(
        &mut self,
        display: &DisplayHandle,
        version: u32,
        global_data: G,
        filter: F,
    ) -> GlobalId
    where
        D: GlobalDispatch<M, G> + Dispatch<M, ()> + 'static,
        M: Resource,
        G: Send + Sync + 'static,
        F: Fn(&Client) -> bool + Send + Sync + 'static,
    { ... }
}
```

### 6.3 Screencopy Handler Design

```rust
pub trait ImageCopyCaptureHandler: ImageCaptureSourceHandler {
    fn image_copy_capture_state(&mut self) -> &mut ImageCopyCaptureState;

    /// Return buffer constraints for the given source, or None to reject.
    fn capture_constraints(&mut self, source: &ImageCaptureSource) -> Option<BufferConstraints>;

    /// Return cursor buffer constraints, or None if cursor capture not supported.
    fn cursor_capture_constraints(&mut self, source: &ImageCaptureSource) -> Option<BufferConstraints> {
        let _ = source;
        None  // default: cursor capture disabled
    }

    /// Called when a new capture session is created.
    fn new_session(&mut self, session: Session);

    /// Called when a new cursor session is created.
    fn new_cursor_session(&mut self, session: CursorSession) {
        let _ = session;
    }

    /// Called when a frame capture is requested.
    /// Compositor should render to the frame's buffer and call `frame.success()` or `frame.fail()`.
    fn capture_frame(&mut self, session: &SessionRef, frame: Frame);

    /// Called when a cursor frame capture is requested.
    fn capture_cursor_frame(&mut self, session: &CursorSessionRef, frame: Frame) {
        frame.fail(FailureReason::Unknown);  // default: fail
    }

    fn session_destroyed(&mut self, session: SessionRef) { let _ = session; }
    fn cursor_session_destroyed(&mut self, session: CursorSessionRef) { let _ = session; }
}
```

---

## 7. Implementation Roadmap

### Phase 1: Image Capture Source (Foundation)

1. **Refactor existing code** at `src/wayland/image_capture_source/mod.rs`:
   - Replace `CaptureSourceType` enum with `UserDataMap` approach
   - Add callback-based handler trait
   - Add `register_custom_source_manager()` for extensions

2. **Ensure selective capability:**
   - `new()` → output only
   - `new_with_toplevel_capture()` → output + toplevel
   - Custom managers via `register_custom_source_manager()`

### Phase 2: Image Copy Capture (Screencopy)

1. **Port cosmic-comp's screencopy.rs:**
   - Replace `ImageCaptureSourceData` with `ImageCaptureSource`
   - Convert to Smithay handler pattern
   - Add module documentation

2. **Key types to implement:**
   - `ImageCopyCaptureState`
   - `Session`, `SessionRef` (RAII ownership)
   - `CursorSession`, `CursorSessionRef`
   - `Frame`, `FrameRef`
   - `BufferConstraints`, `DmabufConstraints`

### Phase 3: Anvil Integration

1. Implement handlers in `anvil/src/state.rs`
2. Test with `grim` or custom client

### Phase 4: Documentation & PR

1. Module-level documentation with examples
2. Changelog entries
3. Draft PR for maintainer review

---

## 8. Licensing and Attribution

### 8.1 License Status

- **Smithay:** MIT License
- **cosmic-comp:** GPL-3.0 License

### 8.2 Re-licensing Permission

ids1024's statement in issue #1900:
> "the image-copy code there is all written by @Drakulix and me at System76, and I don't think there should be any problem with re-licensing that part for inclusion in Smithay"

### 8.3 Required Steps

Before merging any code derived from cosmic-comp:

1. **Formal acknowledgment** from both authors (@Drakulix and @ids1024) that the specific code can be re-licensed under MIT
2. **Attribution** in commit messages and/or code comments
3. **Changelog** noting the contribution origin

### 8.4 Suggested Attribution Format

```rust
// This implementation is based on cosmic-comp's screencopy protocol implementation,
// originally written by Victoria Brekenfeld (@Drakulix) and Ian Douglas Scott (@ids1024)
// at System76, and re-licensed from GPL-3.0 to MIT for inclusion in Smithay.
// Original source: https://github.com/pop-os/cosmic-comp/blob/master/src/wayland/protocols/screencopy.rs
```

---

## 9. Suggested Communication

### 9.1 GitHub Issue Response

I suggest posting the following response to issue #1900:

---

> Thank you for the detailed feedback, @ids1024. The extensibility requirements make complete sense - having compositors define their own capture source types without modifying Smithay is exactly the right approach.
>
> I've analyzed the cosmic-comp implementation and Smithay's existing patterns (particularly `SeatHandler` with associated types and `DmabufHandler` with callbacks). Based on your requirements, I'm proposing a **callback-based design** where:
>
> 1. `ImageCaptureSource` is an opaque handle with a `UserDataMap` for compositor-specific data
> 2. Handler callbacks (`output_source_created`, `toplevel_source_created`) notify the compositor when sources are created, allowing it to store its representation
> 3. A `register_custom_source_manager()` method allows compositors to add custom globals (like workspace capture) that create the same `ImageCaptureSource` type
> 4. The screencopy handler receives `ImageCaptureSource` and calls back to the compositor for constraints and capture
>
> This approach:
> - Avoids generic parameters that would complicate the API
> - Gives compositors full control over source representation
> - Follows Smithay's existing patterns (similar to `DmabufHandler::dmabuf_imported`)
> - Allows COSMIC to keep using its workspace capture extension
>
> I have an immediate need for these protocols and am ready to implement this. Before I proceed:
>
> 1. Does this design direction align with your expectations?
> 2. For the screencopy implementation, I'd like to adapt cosmic-comp's `screencopy.rs` as a foundation. Could you and @Drakulix confirm the re-licensing to MIT is acceptable? I'll include proper attribution.
>
> I'm happy to iterate on the design or discuss alternatives. My goal is to contribute something that serves both Smithay and downstream compositors well.

---

### 9.2 Communication Principles

1. **Show you've done the work:** Reference specific code, patterns, and analysis
2. **Demonstrate respect:** Acknowledge their expertise and existing implementation
3. **Be collaborative:** Ask for confirmation rather than assuming
4. **Express availability:** Make clear you're ready to do the implementation work
5. **Don't over-engineer:** Present the simplest design that meets requirements

---

## Appendix A: Reference Files

The following reference files have been saved locally:

- `/home/greg/smithay/cosmic-comp-reference/screencopy.rs` - cosmic-comp's screencopy implementation
- `/home/greg/smithay/cosmic-comp-reference/image_capture_source.rs` - cosmic-comp's source implementation

---

## Appendix B: Smithay Protocols Studied

| File | Lines | Key Pattern Learned |
|------|-------|---------------------|
| `src/wayland/drm_syncobj/mod.rs` | 540 | Clean modern pattern, GlobalData filtering |
| `src/wayland/foreign_toplevel_list/mod.rs` | 538 | Handle pattern (Arc<Mutex<Inner>>), UserDataMap |
| `src/wayland/dmabuf/mod.rs` | 1227 | Complex buffer handling, ImportNotifier pattern |
| `src/input/mod.rs` | 200+ | Associated type pattern for extensibility |

---

*End of Analysis Document*
