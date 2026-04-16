# Mosaic Pipeline Architecture

This repository defines a scalable architecture for generating, assembling, and rendering very large image mosaics using a staged pipeline built around procedural layout, distributed job orchestration, GPU-based tile selection, USD scene composition, and Unreal Engine rendering.

The system is intentionally separated into two major domains:

1. **Offline / build-time mosaic generation**
2. **Real-time interaction and visual response inside Unreal Engine**

The offline side is responsible for turning a source image and a tile library into an instanced mosaic representation that can scale beyond what would be practical with naive per-object authoring. The runtime side is responsible for responding to motion or tracking input without CPU-side iteration over every instance.

---

## Repository Purpose

This repository is an **architecture repository**, not just a code drop.

It exists to document:

- how the mosaic generation pipeline is decomposed across tools and execution stages
- where data changes representation as it moves through the system
- which responsibilities belong to Houdini, PDG, GPU compute, USD, and Unreal
- how the real-time interaction path is kept GPU-driven inside Unreal
- how the end-to-end system should be reasoned about before implementation is finalized

In other words, this repo is meant to answer:

- **What are the major subsystems?**
- **What does each stage produce?**
- **Why is the pipeline split this way?**
- **Where does scaling come from?**
- **How does runtime interaction avoid collapsing under instance count?**

---

## System Summary

At a high level, the pipeline works like this:

1. A source image and tile dataset are ingested.
2. Houdini generates the procedural mosaic layout and chunk boundaries.
3. PDG distributes chunk-level work into jobs.
4. A GPU matching stage determines the best tile assignment for mosaic regions.
5. Instance data is generated from matching results.
6. USD files are emitted per chunk.
7. Chunked USD outputs are aggregated into a final scene representation.
8. Unreal loads and renders the resulting instanced scene.
9. At runtime, tracking input drives shader-side response through Unreal systems.

This sequence is described in the core architecture document and mirrored by the repository’s Mermaid diagrams. fileciteturn0file2

---

## Major Architectural Principles

### 1. Stage Separation

Each major tool in the stack has a distinct role:

- **Houdini** defines procedural spatial structure.
- **PDG** handles task distribution and orchestration.
- **GPU compute** performs the expensive tile-matching work.
- **USD** acts as the scene interchange and aggregation layer.
- **Unreal Engine** is the runtime renderer and interaction host.

This separation keeps the system modular and allows heavy preprocessing to occur outside the runtime renderer. The current README only states the tool chain, while the architecture docs make clear that each stage owns a specific transformation in the pipeline. fileciteturn0file0 fileciteturn0file2

### 2. Chunked Scalability

The pipeline is designed around **chunking** rather than attempting to solve the entire mosaic as a single monolithic job. Chunking enables:

- parallel dispatch through PDG
- bounded GPU workloads
- per-chunk USD export
- later aggregation into a final scene

This is the core scaling mechanism for very large mosaics.

### 3. Instancing Instead of Unique Geometry

The output is not a giant scene made of individually authored unique meshes. The architecture is built around **instance generation**, with USD outputs centered on structures such as **PointInstancer** and chunked USD files. fileciteturn0file2

### 4. Runtime Interaction Must Stay GPU-Driven

The interaction architecture explicitly avoids CPU iteration over all instances. Tracking input is converted in Unreal, pushed through a Material Parameter Collection, and resolved in material/shader logic. The interaction document is clear that the runtime path is intended to remain GPU-driven. fileciteturn0file1

---

## Systems Included

## 1. Core Pipeline

The core pipeline covers the transformation from source content to renderable mosaic scene.

**Pipeline:**
Houdini → PDG → GPU → USD → Unreal

### Responsibilities by Stage

#### Houdini

Houdini is responsible for defining the procedural mosaic layout. In the current architecture, this includes generating the grid and preparing chunkable spatial structure from the input image.

#### PDG

PDG distributes the work. Rather than treating the mosaic as a single blocking process, PDG turns chunk-level work into tasks that can be dispatched to downstream compute stages.

#### GPU Matching

The GPU stage is where tile matching happens at scale. The Mermaid diagram indicates a pixel/tile comparison loop with repeated distance evaluation and best-match updates until a final tile ID is chosen. This implies the GPU stage is the computational center of the matching system. 

#### Instance Generation

After tile IDs are resolved, the system maps them into prototypes, transforms, and final instance records.

#### USD Export

The architecture then serializes chunk outputs into USD. The current docs identify outputs such as:

- prototype definitions
- PointInstancer data
- positions / indices / transforms
- chunked USD files

#### Aggregation

Chunk-level USD outputs are then stacked or merged into a final USD scene representation.

#### Unreal Rendering

Unreal consumes the USD-produced instance data, applies materials and lighting, and renders the final result.

---

## 2. Interaction System

The interaction system is separate from the offline build pipeline.

**Pipeline:**
Tracking → Actor → Material Parameter Collection → Material → Visual Response

The runtime interaction architecture states that real-time interaction is driven entirely inside Unreal. Tracking input is converted into world-space context, a Blueprint or actor updates a Material Parameter Collection, and the material computes the response per instance. The listed visual effects currently include:

- displacement
- ripple
- reveal
- distortion

The important architectural constraint is that this behavior is **not** meant to iterate over instances on the CPU. It is designed to remain GPU-driven at runtime. fileciteturn0file1

---

## End-to-End Flow

A more explicit end-to-end description of the system is:

### Phase A — Source Preparation

- ingest source image
- ingest tile dataset / tile library
- define mosaic domain and layout rules

### Phase B — Spatial Decomposition

- generate mosaic grid in Houdini
- divide the problem into chunks or regions
- prepare chunk metadata for distributed execution

### Phase C — Distributed Execution

- PDG converts chunks into tasks
- tasks dispatch GPU jobs
- each GPU job evaluates candidate tiles for assigned image regions

### Phase D — Match Resolution

- compute distance or similarity metric for candidates
- update best-match result when a better tile is found
- output resolved tile IDs for the chunk

### Phase E — Scene Representation

- convert tile IDs into prototypes and transforms
- emit per-chunk instance data
- write chunk USD outputs
- aggregate all chunk USD files into a final scene structure

### Phase F — Runtime Rendering

- Unreal loads the final USD-derived instance representation
- materials and lighting are applied
- final mosaic is rendered in-engine

### Phase G — Runtime Interaction

- tracking input enters Unreal
- actor/Blueprint updates MPC values
- materials compute localized response
- user-visible effects appear without CPU-side per-instance updates

---

## Data Products and Outputs

Based on the current architecture files, the primary outputs of the system include:

- chunk-level task definitions
- GPU matching results
- resolved tile IDs
- prototype references
- transforms / positions / indices
- chunked USD files
- aggregated final USD scene
- Unreal-rendered mosaic output

The core architecture document specifically identifies **USD PointInstancer** and **chunked USD files** as key outputs. fileciteturn0file2

---

## File and Document Structure

Current top-level repository contents:

- `architecture_core.md` — core offline generation pipeline
- `architecture_interaction.md` — Unreal runtime interaction architecture
- `diagrams/` — standalone Mermaid diagrams describing subsystem flow

The existing README mentions these files, but it does not explain how they should be read. A better reading order is:

1. `architecture_core.md`
2. `architecture_interaction.md`
3. diagram set from narrow subsystem views to full-system view

Recommended diagram reading order:

1. `diagram_01_core_flow.mmd` — high-level core pipeline
2. `diagram_02_pdg.mmd` — chunk to task to GPU job flow
3. `diagram_03_gpu.mmd` — tile matching loop
4. `diagram_04_instances.mmd` — tile ID to instance construction
5. `diagram_05_usd.mmd` — USD structural layout
6. `diagram_06_aggregation.mmd` — chunk aggregation
7. `diagram_07_unreal.mmd` — Unreal render path
8. `diagram_08_interaction.mmd` — runtime interaction flow
9. `diagram_09_full_system.mmd` — combined architectural summary

---

## Diagram Intent

The Mermaid diagrams in this repository are intentionally separated into standalone files rather than embedded inline into the markdown documents. This is useful for:

- keeping each diagram focused on a single subsystem
- allowing independent revision of architecture visuals
- making the documentation easier to maintain as the design evolves
- supporting architecture reviews where individual stages need to be discussed in isolation

Each diagram captures a different transformation boundary:

- **core flow** shows the major system stages
- **PDG** shows orchestration and dispatch
- **GPU** shows matching logic
- **instances** shows conversion from match result to instance representation
- **USD** shows scene-structure layout
- **aggregation** shows chunk composition
- **Unreal** shows render consumption
- **interaction** shows the runtime control path
- **full system** shows the complete relationship between build-time and runtime systems

---

## Runtime / Build-Time Boundary

One of the most important distinctions in this repository is the separation between:

- **build-time generation** of the mosaic representation
- **runtime rendering and interaction** in Unreal

This matters because large-scale tile selection and scene construction are expensive and should be performed upstream, while the runtime system should focus on:

- loading the prepared representation efficiently
- rendering the result
- handling interaction with minimal CPU overhead

That boundary is one of the key architectural decisions in this design.

---

## Current Implementation Status

This repository appears to be at the **architecture-definition stage** rather than the final production-implementation stage.

What is clearly defined:

- the major subsystems
- the sequence of pipeline stages
- the scaling approach through chunking and distribution
- the representation strategy through instancing and USD
- the Unreal-side interaction model

What is explicitly still pending or not fully captured in the current README:

- detailed data schemas and contracts between stages
- exact GPU matching metric and implementation details
- concrete PDG task definitions and scheduling policy
- exact USD schema/layout choices beyond the current structural summary
- the full Unreal Material Graph implementation

The current README itself notes that **Material Graph is still required for full implementation**. fileciteturn0file0

---

## What This README Should Help a New Engineer Understand

A new engineer opening this repository should be able to answer the following after reading this README:

- what the system is for
- where each tool fits
- how the pipeline scales
- what artifacts are produced at each stage
- how chunking, GPU matching, instancing, USD, and Unreal relate
- why runtime interaction is handled inside Unreal and not by CPU-side iteration
- which parts are architectural design versus implementation detail

If a README does not answer those questions, it is too thin for a repository like this.

---

## Recommended Next Documentation Additions

To make this repository implementation-ready, the next useful additions would be:

### 1. Data Contracts

Define explicit structures for:

- chunk metadata
- GPU input buffers
- tile match output format
- instance records
- USD export payloads
- Unreal import assumptions

### 2. Execution Model

Document:

- batch sizing
- PDG task granularity
- retry / failure behavior
- expected throughput boundaries
- where deterministic reproducibility matters

### 3. Matching Specification

Document:

- tile comparison metric
- color/error distance logic
- tie-breaking behavior
- whether spatial coherence or neighborhood constraints are applied

### 4. Unreal Runtime Specification

Document:

- actor responsibilities
- MPC parameter naming
- material graph inputs/outputs
- supported interaction effects
- performance constraints for high instance counts

### 5. Implementation Status Matrix

Add a simple table that marks each subsystem as one of:

- conceptual
- partially implemented
- validated
- production-ready

---

## Notes

- No icons are used anywhere.
- All diagrams are standalone Mermaid files.
- The architecture separates offline generation from runtime interaction.
- The runtime interaction path is designed to remain GPU-driven.
- Material Graph work is still required to complete the real-time implementation path. fileciteturn0file0turn0file1turn0file2
