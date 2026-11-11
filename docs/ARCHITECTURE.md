# SOFIA architecture

## 1. Target

The primary machine family is the IBM PC-compatible line. The compatibility target is DOS software, including conventional DOS games and applications.

The project should prefer a coherent, dependable IBM PC/DOS environment over emulating every machine family supported by DOSBox-X.

## 2. Design principles

1. Preserve DOS compatibility before optimizing for a smaller codebase.
2. Keep XBRZ available as a supported scaler.
3. Exclude a subsystem only after identifying its source files, dependencies, build switches, and shared code.
4. Avoid broad deletions where a compile-time option or isolated build target is safer.
5. Keep changes small enough to test and bisect.

## 3. Proposed subsystem boundaries

| Subsystem | Initial decision | Required investigation |
|---|---|---|
| IBM PC machine emulation | Keep | Identify supported machine types and defaults |
| DOS kernel / DOS services | Keep | Map compatibility-sensitive code |
| Video and rendering | Keep | Verify XBRZ dependency and build path |
| PC-98 | Exclude from SOFIA profile | Find isolated sources, resources, and shared dependencies |
| MT-32 / Munt | Exclude from SOFIA profile | Trace MIDI backend and optional library linkage |
| Networking | Exclude from SOFIA profile | Trace packet, NE2000, modem, socket, and configuration paths |
| Other non-IBM machine families | Review individually | Determine whether code is isolated or shared |

These are project policy decisions, not claims that the upstream code has already been removed.

## 4. Configuration strategy

Prefer a SOFIA-specific build configuration and narrow compile-time feature switches where upstream supports them. Avoid changing default behavior globally until compatibility tests demonstrate that the change is safe.

## 5. Validation

For every exclusion:
- Build with the feature disabled.
- Confirm no unresolved symbols or accidental dependency remains.
- Run a representative DOS boot and application/game smoke test.
- Verify video output and XBRZ.
- Record the exact upstream commit and build environment.
