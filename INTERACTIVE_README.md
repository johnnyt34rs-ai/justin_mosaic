# Interactive Architecture Package

This package contains the Interactive Architecture for the mosaic system.

## Contents

- `interactive_architecture.md`
  - main architecture document
  - contains no embedded Mermaid blocks
  - references each diagram as a separate `.mmd` file

- `diagrams/`
  - standalone Mermaid source files
  - one diagram per file

## Files

- `diagrams/IA-001_system_context.mmd`
- `diagrams/IA-002_runtime_layers.mmd`
- `diagrams/IA-003_unreal_runtime_structure.mmd`
- `diagrams/IA-004_blueprint_architecture.mmd`
- `diagrams/IA-005_motion_to_effect_flow.mmd`
- `diagrams/IA-006_frame_update_sequence.mmd`

## Notes

- No icons are used anywhere in this package.
- All Mermaid diagrams are externalized.
- The architecture document references diagrams by Model ID and file path.
- The Material Graph is intentionally called out as missing and requiring its own separate specification.
- The SOP Node Graph is intentionally called out as missing and requiring its own separate specification.

## Suggested Next Files

The following should be produced next:

1. Material Graph Specification
2. SOP Node Graph Specification
3. Tracking Input Specification
4. Blueprint Contract Specification

