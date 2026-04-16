# Interaction Architecture

## Overview

Real-time interaction is driven entirely inside Unreal.

## Flow

Tracking input is converted to world space.
Blueprint updates Material Parameter Collection.
Material computes per-instance response.

## Components

- Tracking Actor
- Material Parameter Collection
- Tile Material
- HISM instances

## Effects

- Displacement
- Ripple
- Reveal
- Distortion

## Notes

- No CPU iteration over instances
- All interaction is GPU-driven
