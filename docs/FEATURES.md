# Feature policy

This document records the intended SOFIA profile. It does not claim that the corresponding upstream code has already been removed.

## Keep

- IBM PC-compatible machine emulation.
- DOS compatibility and core DOS services.
- Standard PC video, input, storage, and audio paths needed by DOS software.
- XBRZ image scaling.

## Exclude from the initial SOFIA profile

- PC-98-specific machine support.
- MT-32 / Munt emulation and its optional integration.
- Networking features and backends.

## Review before deciding

Do not automatically remove a component merely because it is non-standard or uncommon. First establish whether it is:
- specific to a separate machine family;
- an optional backend with a clean boundary;
- shared by ordinary IBM PC/DOS functionality; or
- required by a dependency used elsewhere.

## Explicit non-goals for the first pass

- Replacing the DOSBox-X core.
- Rewriting the renderer.
- Removing general MIDI, Sound Blaster, Gravis Ultrasound, or other ordinary DOS audio paths without a separate compatibility decision.
- Trading compatibility for code-size reduction without measurements.
