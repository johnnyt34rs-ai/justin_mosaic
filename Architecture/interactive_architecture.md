# Interactive Architecture

## Mosaic Interaction System (Unreal Runtime Interaction Layer)

---

## 1. System Definition

This architecture defines the interactive runtime layer for the mosaic generation system. It sits on top of the Core Architecture and governs how a person in front of the installation influences the mosaic while it is running inside Unreal.

The purpose of this document is to guide a VFX-oriented implementer through the runtime system without assuming deep software architecture experience.

This architecture covers:

- person detection and motion acquisition
- runtime interpretation of motion data
- transformation of motion into artistic control signals
- Blueprint-driven orchestration inside Unreal
- material and geometry response at runtime
- safe separation between offline generation and live interaction

This architecture does **not** replace the Core Architecture. The Core Architecture remains the authoritative source for:

- mosaic generation
- tile assignment
- chunk processing
- GPU matching
- USD representation
- base scene assembly

The Interactive Architecture extends that system by adding a controlled runtime behavior layer.

---

## 2. Design Intent

The interactive system must satisfy six requirements:

1. A person can approach and move in front of the mosaic.
2. The system can detect that motion and convert it into stable runtime data.
3. Unreal can use that data to drive visible change in the mosaic.
4. The effect must remain artistically controllable.
5. The runtime layer must not destroy or invalidate the offline-generated mosaic.
6. The system must be understandable and maintainable by a VFX-oriented builder.

The central idea is simple:

**offline systems decide what the mosaic is; runtime systems decide how the mosaic behaves when someone interacts with it.**

---

## 3. Scope of This Architecture

This document includes:

- runtime interaction architecture
- Unreal subsystem layout
- Blueprint control model
- motion-to-effect mapping
- data contracts between sensing and rendering
- update sequencing
- practical implementation guidance

This document intentionally excludes full implementation of:

- SOP node graph specification
- Material Graph specification
- final computer vision model selection
- final hardware bill of materials
- final latency profiling results

Those items must exist as separate technical documents.

---

## 4. Relationship to Core Architecture

The complete system now has two top-level architecture documents:

1. **Core Architecture**
   - Houdini -> PDG -> GPU -> USD -> Unreal
   - responsible for producing the mosaic as data and geometry

2. **Interactive Architecture**
   - sensor/input -> Unreal runtime -> Blueprint/material/instance response
   - responsible for live behavior when a person interacts with the mosaic

The runtime layer must treat the offline-generated mosaic as a stable base asset.

### Reference Diagram

- **Model ID:** IA-001
- **File:** `diagrams/IA-001_system_context.mmd`
- **Purpose:** shows where the Interactive Architecture attaches to the Core Architecture

---

## 5. Architectural Principle

The interaction layer is built around a strict separation of concerns.

### 5.1 Offline Domain

The offline domain creates the mosaic.

It includes:

- image analysis
- tile matching
- chunk generation
- USD export
- scene assembly

This domain is heavy, deterministic, and not intended to run per viewer movement.

### 5.2 Runtime Domain

The runtime domain responds to a person.

It includes:

- sensing
- motion interpretation
- control signals
- Blueprint orchestration
- material parameter updates
- instance animation and effect triggering

This domain is lightweight, frame-driven, and must remain stable under continuous input.

### Rule

The runtime domain may **modulate** the mosaic, but it must not recompute the full mosaic.

That means:

- no re-running tile matching in response to a viewer
- no full-scene rebuilds per interaction
- no destructive mutation of the authored source mosaic

Instead, the runtime system should operate through controlled parameters such as:

- visibility
- offset
- scale
- color intensity
- emissive response
- local reveal masks
- ripple fields
- attraction or repulsion fields

---

## 6. Runtime Layer Breakdown

The Interactive Architecture is composed of five layers.

### 6.1 Sensing Layer

Responsibility:
Acquire information about a person or people interacting with the installation.

Possible inputs:

- depth camera
- RGB camera with body tracking
- LiDAR-style occupancy feed
- simplified blob tracking
- prerecorded test input for development

Minimum required output:

- subject presence
- subject position in installation space
- motion vector
- optional body landmarks

Important guidance for the VFX implementer:

Start with the simplest stable input. A clean tracked centroid is more valuable than an unstable full skeleton.

### 6.2 Interpretation Layer

Responsibility:
Convert raw sensing data into artistically useful control values.

Example derived signals:

- nearest point of approach
- speed magnitude
- lateral movement
- dwell time
- interaction zone occupancy
- motion energy
- direction of travel

This layer protects the visual system from noisy input.

Typical operations:

- smoothing
- thresholding
- temporal averaging
- dead zones
- hysteresis
- normalization into 0..1 control values

### 6.3 Control Layer

Responsibility:
Translate interpreted signals into a small set of explicit runtime controls.

Examples:

- `reveal_strength`
- `ripple_radius`
- `motion_influence`
- `focus_position`
- `distortion_amount`
- `color_shift`
- `tile_lift`
- `emissive_gain`

This is the most important simplification in the system.

The rest of Unreal should respond to a curated control set, not directly to raw tracking data.

### 6.4 Orchestration Layer

Responsibility:
Coordinate runtime execution inside Unreal.

This is where Blueprint, Unreal subsystems, and actor-level logic operate.

Key duties:

- receive interaction data
- update control values
- dispatch events to visual systems
- maintain order of operations per frame
- expose artist-friendly tuning controls

### 6.5 Response Layer

Responsibility:
Produce the final visible behavior.

This includes:

- material parameter changes
- instance transforms
- Niagara-based secondary effects if needed
- local mesh response
- reveal, pulse, or wave behavior across the mosaic

### Reference Diagram

- **Model ID:** IA-002
- **File:** `diagrams/IA-002_runtime_layers.mmd`
- **Purpose:** shows the five-layer runtime model from sensing through visual response

---

## 7. Recommended Unreal Runtime Structure

For a VFX-oriented implementation, the cleanest runtime layout is:

### 7.1 Input Adapter

A dedicated component or actor that receives data from the tracking source.

Responsibilities:

- connect to external or internal tracking feed
- convert coordinates into Unreal space
- package data into a consistent runtime struct

Recommended output struct fields:

- `subject_id`
- `is_present`
- `world_position`
- `velocity`
- `speed`
- `confidence`
- `timestamp`

### 7.2 Interaction Manager

A central Unreal actor or subsystem that owns all interpreted runtime state.

Responsibilities:

- filter raw subject data
- maintain active subject list
- compute derived control signals
- expose values to Blueprint and materials

This should be the single source of truth for live interaction state.

### 7.3 Mosaic Controller Blueprint

This is the main Blueprint-facing orchestration layer.

Responsibilities:

- read current interaction controls each frame or on update
- push values into material parameter collections
- send instance transform offsets where required
- trigger effect timelines
- manage zones, reveals, pulses, and response states

### 7.4 Visual Response Components

These are the systems that actually change what the viewer sees.

They may include:

- Material Parameter Collection updates
- per-instance custom data
- Niagara systems
- local transform offset logic
- reveal masks
- render target driven displacement fields

### 7.5 Debug and Tuning Layer

A VFX implementer needs runtime visibility.

Required debug features:

- display tracked position
- display interaction zones
- display normalized control values
- enable or disable smoothing
- replay recorded motion input
- freeze current input for lookdev

### Reference Diagram

- **Model ID:** IA-003
- **File:** `diagrams/IA-003_unreal_runtime_structure.mmd`
- **Purpose:** shows the recommended Unreal-side runtime object structure

---

## 8. Blueprint Architecture

The Blueprint system should be used as the artist-facing orchestration layer, not as the place where all math and all data handling live.

### 8.1 Blueprint Responsibilities

Blueprint should:

- read curated control values
- sequence effects
- drive timelines
- manage artist-exposed parameters
- coordinate response events
- support quick iteration and look development

### 8.2 Blueprint Should Not Own

Blueprint should not be the primary home for:

- raw sensor parsing
- unstable coordinate conversion
- heavy per-point CPU loops over millions of elements
- low-level transport logic

### 8.3 Recommended Blueprint Split

Use three Blueprint tiers:

#### A. Interaction Manager Blueprint Interface

Purpose:
Provide read access to current runtime control values.

#### B. Mosaic Master Controller Blueprint

Purpose:
Own the global visual state of the mosaic.

Typical jobs:

- set global material parameters
- drive reveal radius
- drive pulse intensity
- define effect state transitions
- select visual mode

#### C. Zone or Effect Blueprints

Purpose:
Handle local responses or specialized visual behaviors.

Examples:

- center attraction effect
- edge distortion effect
- proximity glow effect
- motion trail generator

### 8.4 Artist Control Requirements

Expose clean controls such as:

- response gain
- smoothing amount
- minimum activation speed
- reveal falloff
- pulse decay
- distortion scale
- color blend amount
- interaction zone radius

These controls should be grouped clearly in the Unreal editor.

### Reference Diagram

- **Model ID:** IA-004
- **File:** `diagrams/IA-004_blueprint_architecture.mmd`
- **Purpose:** shows how Blueprint reads the control layer and distributes visual commands

---

## 9. Motion-to-Effect Mapping

This is the artistic heart of the Interactive Architecture.

The system should not ask, "What exact body motion occurred?" It should ask, "What visual response should this motion produce?"

### 9.1 Input Signals

Useful input signals include:

- subject distance from mosaic center
- motion speed
- direction of travel
- dwell time in a region
- occupancy count
- acceleration spikes

### 9.2 Derived Effect Controls

Map those inputs to a smaller set of artistic outputs.

Examples:

- slow motion near the center -> soft reveal and glow
- fast lateral motion -> directional ripple or tile sweep
- prolonged dwell -> local sharpening or color enrichment
- multiple people -> broader disturbance field
- sudden motion spike -> pulse burst

### 9.3 Recommended Mapping Strategy

Do not wire a single input directly to a single dramatic effect unless the behavior is intentionally simple.

A better approach is:

1. normalize input
2. smooth over time
3. apply thresholds
4. remap to response curve
5. combine with artistic gain values
6. drive one or more effect parameters

### 9.4 Stability Requirement

Without stability controls, the mosaic will look noisy and amateur.

The interaction system must include:

- smoothing
- activation thresholds
- cooldown or decay
- optional hysteresis

### Reference Diagram

- **Model ID:** IA-005
- **File:** `diagrams/IA-005_motion_to_effect_flow.mmd`
- **Purpose:** shows how motion data becomes a stable artistic response

---

## 10. Data Contracts

The VFX implementer will benefit from thinking in explicit data contracts.

### 10.1 Tracking Contract

The tracking provider must output a stable subject record.

Minimum contract:

- subject identifier
- position in world or calibrated local space
- velocity or frame-to-frame motion delta
- confidence value
- timestamp

### 10.2 Interaction Contract

The Interaction Manager must output curated control data.

Recommended contract:

- focus position
- normalized motion energy
- reveal strength
- ripple radius
- interaction active flag
- active subject count
- visual mode

### 10.3 Rendering Contract

The visual systems must consume runtime values in a way that does not require full scene reconstruction.

Recommended output pathways:

- Material Parameter Collection
- Dynamic Material Instance parameters
- per-instance custom data
- Niagara user parameters
- render targets for field-based effects

### Guidance

If a value must be updated every frame, it should live in a runtime-friendly pathway.
If a value affects the authored mosaic structure, it belongs in the offline domain instead.

---

## 11. Material Graph Note

The Material Graph is required but is **not defined in this document**.

A separate Material Graph specification must be created.

That document should define:

- base mosaic shading model
- how interaction fields enter the material
- emissive logic
- color modulation logic
- reveal masking logic
- distortion or depth cue logic
- any use of render targets or distance fields

This architecture assumes the Material Graph will be able to consume runtime control values but does not prescribe the full shader graph.

---

## 12. SOP Node Graph Note

The SOP node graph is also required but is **not defined in this document**.

A separate SOP Node Graph specification must be created.

That document should define:

- point generation
- chunk creation
- attribute preparation
- tile placeholder assignment
- animation bake strategy if needed
- export packaging for Unreal or USD

This Interactive Architecture depends on the SOP graph only insofar as it requires a stable base mosaic representation to exist before runtime interaction begins.

---

## 13. Update Model

The runtime system should follow a disciplined update sequence.

### Per-frame sequence

1. acquire latest tracking data
2. validate and smooth input
3. compute derived interaction controls
4. update Blueprint-accessible state
5. push values into materials, instances, and effects
6. render frame

This order matters.

If rendering reads partially updated interaction state, the effect will look unstable.

### Reference Diagram

- **Model ID:** IA-006
- **File:** `diagrams/IA-006_frame_update_sequence.mmd`
- **Purpose:** shows the recommended frame update ordering

---

## 14. Implementation Guidance for a VFX Builder

This section is deliberately practical.

### Start in this order

1. load the offline-generated mosaic in Unreal
2. create a fake interaction source using a test actor or mouse-driven proxy
3. build the Interaction Manager
4. expose a small set of runtime controls
5. connect those controls to one simple visual response
6. add debug overlays
7. only then connect real tracking input

### Why this matters

If real tracking is connected too early, every visual problem becomes difficult to diagnose.

By starting with a fake input source, the VFX implementer can prove that:

- control mapping works
- material response works
- Blueprint orchestration works
- update order is correct

### First proof-of-function target

The first successful interactive prototype should do only this:

- detect a single interaction point
- compute distance to that point
- drive a circular reveal or glow in the mosaic

Do not start with skeletons, multiple people, and complex waves.

Start with one stable effect that proves the pipeline is correct.

---

## 15. Performance Guidance

The runtime interaction layer must stay lightweight.

### Recommended rules

- prefer global parameters over per-instance updates when possible
- use per-instance custom data only where the effect truly requires it
- avoid CPU iteration over millions of instances per frame
- push field-style logic into materials when appropriate
- treat Niagara as secondary polish, not the primary control system
- profile early once the first effect works

### Practical interpretation

If the visual result can be expressed as a field sampled in the material, that is often better than trying to individually touch every tile at runtime.

Examples of efficient runtime drivers:

- radial distance from a focus point
- screen-space mask
- world-space falloff field
- render target driven interaction map

---

## 16. Failure Modes to Avoid

A VFX implementation can go wrong in predictable ways.

### Common failure modes

- direct wiring of noisy tracking input into visible effects
- rebuilding or reauthoring geometry during interaction
- placing too much logic inside one giant Blueprint
- using too many artist parameters with no grouping
- per-frame CPU work across too many instances
- lack of debug views
- trying to solve sensing, lookdev, and performance simultaneously

### Prevention strategy

- keep tracking, control, and response separate
- keep the control vocabulary small
- prove one clean effect first
- add debug overlays from the beginning
- document parameter meaning clearly

---

## 17. Minimum Deliverables Required After This Architecture

This document is not the end state. It defines the structure, but the following documents or assets still need to be produced:

1. Material Graph Specification
2. SOP Node Graph Specification
3. Tracking Input Specification
4. Blueprint Variable and Event Contract
5. Unreal Test Scene for interaction development

---

## 18. Final Architecture Statement

The Interactive Architecture should be understood as a controlled runtime response layer attached to a prebuilt mosaic.

The offline system creates the mosaic.
The runtime system interprets a person.
Unreal converts that interpreted motion into artistic behavior.

The correct implementation path is:

- stable offline mosaic first
- curated interaction controls second
- Blueprint orchestration third
- material and instance response fourth
- real tracking integration after the artistic control path is proven

That sequence gives the VFX implementer the highest chance of building something that is both visually strong and technically stable.

---

## 19. Diagram Index

- **IA-001** — `diagrams/IA-001_system_context.mmd`
- **IA-002** — `diagrams/IA-002_runtime_layers.mmd`
- **IA-003** — `diagrams/IA-003_unreal_runtime_structure.mmd`
- **IA-004** — `diagrams/IA-004_blueprint_architecture.mmd`
- **IA-005** — `diagrams/IA-005_motion_to_effect_flow.mmd`
- **IA-006** — `diagrams/IA-006_frame_update_sequence.mmd`

