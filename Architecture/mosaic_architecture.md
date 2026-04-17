# Mosaic Generation System Architecture
## Houdini → PDG → GPU → USD → Unreal

---

## 1. System Definition

This system generates large-scale image mosaics using:

- Procedural scene construction
- Distributed computation
- GPU-accelerated tile matching
- Instanced geometry representation
- Real-time rendering

The system is designed to scale to millions of elements while maintaining:

- Determinism
- Reproducibility
- Performance

---

## 2. Architectural Layers

The system is composed of four strictly separated layers.

---

### 2.1 Authoring Layer (Houdini)

See: [`layer_authoring.mmd`](layer_authoring.mmd)

**Responsibility**
Define the structure of the mosaic and prepare data for distributed execution.

**Key Functions**
- Convert input image into a point grid
- Assign spatial layout (`P`)
- Define chunk boundaries
- Prepare attribute schema

**Core Outputs**
Point cloud representing mosaic cells.

**Attributes**
- `P` (position)
- `chunk_id`
- `cell_uv`
- `tile_id` (placeholder)

**Constraints**
- No brute-force matching
- No global simulation across entire dataset
- Must remain chunkable and deterministic

---

### 2.2 Distribution Layer (PDG)

See: [`layer_distribution.mmd`](layer_distribution.mmd)

**Responsibility**
Orchestrate execution across compute resources.

**Key Functions**
- Partition mosaic into independent chunks
- Generate work items (one per chunk)
- Dispatch jobs to compute nodes
- Track dependencies and outputs

**Work Item Definition**
Each work item includes:
- Chunk bounds
- Subset of points
- Reference to tile dataset

**Execution Model**
- Stateless
- Parallel
- Retry-safe

**Output**
Processed chunk data (tile assignments + transforms)

---

### 2.3 Compute Layer (GPU Cluster)

See: [`layer_compute.mmd`](layer_compute.mmd)

**Responsibility**
Perform tile matching at scale.

**Core Problem**
For each mosaic cell, find the best matching tile based on color similarity.

**Execution Model**
- One GPU thread per cell
- Brute-force comparison across tile dataset

**Data Inputs**
- Cell colors (LAB space)
- Tile colors (LAB space)

**Output**
`tile_id` per cell

**Constraints**
- Memory bandwidth bound
- Must use contiguous buffers (Structure of Arrays)
- Minimize data transfer per job

---

### 2.4 Representation Layer (USD)

See: [`layer_representation.mmd`](layer_representation.mmd)

**Responsibility**
Store the mosaic efficiently for rendering.

**Core Structure**
- Prototypes (tile meshes)
- PointInstancer (instances)

**Components**

*Prototypes*
- One mesh per tile type
- Shared across all instances

*PointInstancer*
Stores:
- Positions (`P`)
- Prototype indices (`protoindex`)
- Orientations (`orient`)
- Scales (`scale`)

**Output Format**
- One USD file per chunk
- Aggregated via layer composition

**Constraints**
- No duplicated geometry
- No per-instance mesh creation
- Must remain streamable

---

### 2.5 Rendering Layer (Unreal Engine)

See: [`layer_rendering.mmd`](layer_rendering.mmd)

**Responsibility**
Render the mosaic at scale.

**Key Functions**
- Import USD data
- Convert to instanced meshes
- Apply material system
- Execute lighting and rendering

**Rendering Model**
- Hierarchical instancing (HISM)
- GPU-driven shading
- Real-time or offline output

**Constraints**
- Avoid per-instance CPU updates
- Minimize draw calls
- Keep material complexity low

---

## 3. End-to-End Data Flow

See: [`mosaic_dataflow.mmd`](mosaic_dataflow.mmd)

---

## 4. Processing Pipeline

1. Input image and tile dataset are prepared
2. Houdini generates a point-based mosaic grid
3. Grid is partitioned into chunks
4. PDG dispatches chunk jobs
5. GPU performs tile matching per chunk
6. Tile IDs are assigned to each cell
7. Instances are constructed
8. USD chunks are exported
9. USD layers are aggregated
10. Scene is imported into Unreal Engine
11. Mosaic is rendered

---

## 5. Core Design Principles

| Principle | Description |
|---|---|
| Separation of Concerns | Authoring, compute, and rendering are independent |
| Chunk-Based Scaling | All processing operates on independent regions |
| GPU Utilization | Heavy computation is isolated to GPU kernels |
| Instancing | Geometry is never duplicated |
| Determinism | Same input produces identical output |

---

## 6. Constraints and Limits

| Domain | Constraint |
|---|---|
| Memory | Tile datasets must fit GPU memory or be batched |
| I/O | Chunking required to avoid large file bottlenecks |
| Rendering | Instance count must be managed via instancing and LOD |

---

## 7. Non-Goals

- Real-time generation in Houdini
- CPU-based tile matching at scale
- Per-instance mesh creation
- Monolithic (non-chunked) processing

---

## 8. Status

This document defines the core architecture only.

The following components are not included:
- SOP node graph specification
- Material graph implementation
- Interaction system

These are separate layers built on top of this foundation.
