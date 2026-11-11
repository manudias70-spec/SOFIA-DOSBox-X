# Build and verification plan

## Current stage: specification scaffold

The repository does not yet contain upstream DOSBox-X source or a SOFIA executable.

## Stage 1 — establish a reproducible upstream base

1. Select and record an upstream DOSBox-X commit.
2. Import the source while preserving license and attribution files.
3. Record compiler, build tools, and supported host platforms.
4. Build an unmodified baseline before applying SOFIA changes.

## Stage 2 — map feature boundaries

For PC-98, MT-32/Munt, and networking, document:
- relevant source directories and files;
- configure/CMake/Make options;
- third-party libraries and bundled code;
- shared code that must remain;
- expected impact on ordinary IBM PC/DOS operation.

## Stage 3 — implement the SOFIA profile

1. Use existing upstream feature switches where possible.
2. Add narrowly scoped SOFIA build options only when necessary.
3. Keep XBRZ enabled and verify it is actually compiled.
4. Avoid unrelated refactoring.

## Stage 4 — test

Minimum smoke-test checklist:
- [ ] Clean build from a documented environment.
- [ ] DOS boot to command prompt.
- [ ] Mount and access a disk image or directory.
- [ ] Keyboard and mouse input.
- [ ] Standard VGA output.
- [ ] XBRZ scaler selection and rendering.
- [ ] Representative DOS game/application launch.
- [ ] Audio initialization and playback.
- [ ] No PC-98, Munt, or network dependency in the SOFIA build.

## Release discipline

Tag only tested builds. Publish build instructions, upstream commit, enabled/disabled feature list, known limitations, and checksums with each release.
