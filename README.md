# SOFIA-DOSBox-X

A focused DOSBox-X build profile aimed at **IBM PC-compatible systems and DOS software**.

SOFIA is intended to keep the parts that serve a practical IBM PC/DOS experience, while avoiding unrelated platform emulation and optional subsystems that add maintenance cost.

## Project goals

- IBM PC-compatible systems as the primary target.
- Broad DOS application and game compatibility.
- Keep the **XBRZ** scaler available.
- Keep the build understandable, reproducible, and maintainable.
- Make exclusions explicit and evidence-based rather than deleting code blindly.

## Initial scope

| Area | Direction |
|---|---|
| IBM PC / compatible machines | In scope |
| DOS compatibility | In scope |
| XBRZ scaler | Keep |
| PC-98-specific support | Out of scope |
| MT-32 / Munt integration | Out of scope |
| Networking | Out of scope for the initial profile |
| Other non-standard machine families | Review individually before exclusion |

See [the architecture notes](docs/ARCHITECTURE.md), [the build plan](docs/BUILD.md), and [the feature policy](docs/FEATURES.md).

## Important status

This repository currently contains the SOFIA project specification and planning scaffold. It is **not yet a compiled custom DOSBox-X binary**. The next engineering phase is to pin an upstream DOSBox-X revision, inspect its build system and feature boundaries, then implement and test exclusions incrementally.

## Upstream

SOFIA is based on the DOSBox-X project. Upstream source and licensing notices must be preserved when the source is imported or adapted. See [UPSTREAM.md](UPSTREAM.md).
