# Upstream audit — first pass

## Audited upstream

- Repository: https://github.com/joncampbell123/dosbox-x
- Branch inspected: `master`
- Candidate baseline commit: `e2b92bb86382a5444ce42157a37138b459ec98b5`
- Upstream `configure.ac` package version: `2026.10.01`
- This is a proposed baseline reference, not yet an imported source tree.

## Confirmed build switches

The upstream `BUILD.md` and `configure.ac` document these relevant options:

| Goal | Existing option | Finding |
|---|---|---|
| Keep XBRZ | `--enable-xbrz` | Supported; documented default is enabled |
| Disable MT-32 | `--disable-mt32` | Supported |
| Disable SDL networking | `--disable-sdlnet` | Supported by configure |
| Disable monitor console | `--disable-monitor` | Supported; separate from all network code |

The configure logic shows that SDL_net / SDL2_net can enable internal modem and IPX support. The monitor console has its own switch and may use TCP when SDL_net is available. These are related but distinct paths; disabling one must not be assumed to remove every networking-related source file.

## XBRZ

The upstream configure logic defines `C_XBRZ` when XBRZ is enabled, and the build guide documents `--enable-xbrz` with a default of yes. SOFIA should explicitly enable it in its build configuration and verify the compiled result.

## PC-98 and non-IBM machine support

This first pass did not establish a safe, complete PC-98 exclusion switch. Do not remove files based on filename guesses. The next audit must trace PC-98 machine initialization, BIOS/ROM resources, video/audio dependencies, configuration options, and any code shared with IBM PC emulation.

## Important distinction

A configure switch can prevent a feature from being built without removing its source from the repository. SOFIA's first implementation should prefer documented switches, then consider source-level removal only after dependency analysis and a successful baseline build.

## Next engineering actions

1. Import or otherwise establish a reproducible source baseline at the candidate commit above.
2. Build the unmodified baseline on the intended host platform.
3. Add a SOFIA build profile that explicitly keeps XBRZ and disables MT-32 and SDL networking.
4. Audit PC-98 separately before excluding it.
5. Run DOS boot, video/scaler, input, storage, and audio smoke tests after each change.

## Sources inspected

- `README.md`
- `BUILD.md`
- `INSTALL.md`
- `configure.ac`
- `Makefile.am`

Upstream source and license notices must be retained when importing code.

## Performance regression investigation: Quake DOS / x87

### User-reported observation

The user reports that early DOSBox-X builds that already included AWE32 emulation delivered approximately 90 FPS in the same Quake DOS use case, while later builds deliver approximately 70 FPS. This is a valuable regression clue, but it is not yet a controlled benchmark: exact build IDs, configuration, host, scene, resolution, scaler, and audio settings still need confirmation.

The AWE32 feature itself is therefore not the first suspect. Preserve AWE32 and its ROM configuration while investigating changes introduced after the faster builds.

### FPU changes found in upstream history

A review of upstream commits around late September and October 2026 shows a substantial x87 refactor beginning before October 10, not just a single change on that date. Relevant commit subjects include:

- 2026-09-28: FPU: extract shared instruction helpers; FPU: use explicit binary64 register type.
- 2026-09-29 to 2026-10-02: multiple instruction-handler unifications and exception-handling changes.
- 2026-10-02: FPU: refactor input exception handling; FPU: handle exceptional arithmetic inputs; FPU: defer EA denormal exceptions; merge of fpu-overhaul.
- 2026-10-03: Wire FISTTP into dynamic CPU cores.
- 2026-10-10: Implement x87 numeric exceptions; Unmask IRQ 13 for configured FPU; Abort unmasked x87 exception paths.

The October 10 commits alter x87 exception behavior, status handling, IRQ 13, and exception delivery. The wider late-September/early-October FPU overhaul is also a candidate because it predates those final exception commits.

### Working hypothesis and caution

The timing and workload make the x87 changes a credible regression candidate for Quake DOS, which is FPU-intensive. However, commit chronology alone does not prove that these changes caused the reported FPS difference. Do not revert the AWE32 implementation or claim a fix without an A/B benchmark.

### Required A/B plan

1. Identify the exact last-known-good build (about 90 FPS) and first-known-slow build (about 70 FPS), if possible.
2. Keep the same Quake DOS executable, save/scene, resolution, scaler (XBRZ), cycles/core settings, AWE32 ROM, sound configuration, and host machine.
3. Build/test the last-known-good revision and later revisions in sequence, changing one revision range at a time.
4. First isolate the FPU-overhaul range (late September through October 3); then isolate the October 10 exception/IRQ changes.
5. Record average FPS and frame-time consistency, plus audio glitches and correctness. Verify x87 behavior with compatibility tests before accepting any optimization.
6. Preserve AWE32 and XBRZ in every candidate build.

No performance test has been run as part of this source-history review. The 90/70 FPS values are user-reported, not independently reproduced.


### New benchmark anchor: October 2, 2026

The user has now supplied a more precise result: the DOSBox-X version dated October 2, 2026 reaches **99.6 FPS** in their Quake DOS test. This is a user-reported measurement; the exact packaged build identifier and test conditions are still not captured.

The upstream history for October 2 contains the merge commit:

- `e013b8bf8e27ef878a646049bf5b2e732878d616` — merge of PR #6629, `fpu-overhaul`, authored/committed 2026-10-02 22:56 UTC.

That day's history already includes the main FPU overhaul work, including `FPU: refactor input exception handling`, `FPU: handle exceptional arithmetic inputs`, and `FPU: defer EA denormal exceptions`. Therefore, the user's 99.6 FPS result makes the broad FPU overhaul itself a weaker explanation for the later slowdown: it was already present in the October 2 upstream history. This does not rule out interactions, build differences, or a different October 2 package revision.

The next narrower candidates are changes after this anchor, especially:

- 2026-10-03: `2b7b3e...` — `Wire FISTTP into dynamic CPU cores` (full upstream commit: `2b7b3f4723644a4b4d92ff62fd464944b0e24b0b`).
- 2026-10-10: `cbf3c4b...` — `Implement x87 numeric exceptions` (full upstream commit: `cbf3c4bb302022afb9f18cbe3e25e8663fb4ce2b`); `3ca845c...` — `Unmask IRQ 13 for configured FPU`; `1609dde...` — `Abort unmasked x87 exception paths`.

Revised investigation order:
1. Treat the user's October 2 build (99.6 FPS) as the provisional last-known-good anchor.
2. Identify the exact source commit/package corresponding to that build before assuming the October 2 merge SHA is its exact revision.
3. Compare that build against the first slower build, first isolating the October 3 dynamic-core FISTTP change, then subsequent x87 exception/IRQ changes.
4. Keep the same Quake scene, resolution, core/cycles, scaler, AWE32 ROM/audio settings, and host. Do not change AWE32 or XBRZ during this investigation.

No independent benchmark or source bisect has yet been performed. The 99.6 FPS figure is user-reported.


### Second confirmed user benchmark point: October 18 build

The user has now re-tested a downloaded October 18 build and reports **75 FPS**, compared with **99.6 FPS** on the downloaded October 2 build. The difference is 24.7% lower relative to the October 2 result. These are user-reported measurements, not independently reproduced benchmarks. Exact asset names, build provenance, host/configuration, and whether the same test scene/settings were held constant remain to be captured.

The upstream history shows an important candidate change after the October 2 anchor:

- 2026-10-11: `Always use dynamic FPU handling` (`dda3ea7b2414eae571f0e5c8aad0c7b36867bddf`)
- 2026-10-11: `Handle coprocessor exceptions in dynamic cores` (`290aa213eca273f14506d387ca35e3d623e52e45`)
- 2026-10-11: `Switch dynamic host FPU for exceptions` (`75ec62b63d358fafa5b9398b57418715ccf191d9`)
- 2026-10-11: `Handle x87 exceptions in dynamic cores` (`47845490cbc5ab584db467839b99514f574fbe46`)

The October 14 development build changelog also includes the October 10–11 x87 numeric-exception and dynamic-core exception changes. The official 2026.10.01 release notes explicitly warn that routing FPU/MMX opcodes through the normal core can make some DOS games slower, and mention a configuration option for DOS games that do not multitask. This makes the dynamic-core/FPU dispatch changes a strong investigation target, but it is not yet proof of causation.

Do not guess or prescribe a configuration override until the exact option semantics are verified against the tested build's source. In particular, `dynamic_nodhfpu` is a distinct core mode and should not be casually equated with “enable FPU in the dynamic core”; the source uses it to call `CPU_Core_Dyn_X86_SetFPUMode(false)`.

Revised next steps:
1. Record exact filenames/version strings for both downloaded builds and confirm the October 18 binary is the same Windows/architecture variant as October 2.
2. Confirm identical Quake executable, benchmark/demo/scene, resolution, XBRZ settings, cycles/core, audio/AWE32 ROM, and host settings.
3. Inspect the October 11 dynamic-core FPU commits and the actual configuration option introduced/documented for the fallback behavior.
4. A/B test the October 18 build with only that verified option changed, preserving AWE32 and XBRZ.
5. If needed, compare builds immediately before and after the October 11 commits.

No independent benchmark, configuration change, or fix has been performed.
