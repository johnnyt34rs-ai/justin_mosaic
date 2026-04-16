# Core Architecture

## Overview

This system generates large-scale mosaics using distributed computation and instancing.

## Pipeline

Houdini defines procedural layout.
PDG distributes chunked workloads.
GPU cluster performs tile matching.
USD stores instanced output.
Unreal renders the final result.

## Steps

1. Input image + tile dataset
2. Generate grid in Houdini
3. Chunk image into regions
4. Dispatch via PDG
5. GPU tile matching
6. Instance generation
7. USD export per chunk
8. USD aggregation
9. Unreal rendering

## Outputs

- USD PointInstancer
- Chunked USD files
