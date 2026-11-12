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


### Selected upstream baseline: DOSBox-X 2026.10.01

The project has selected the official DOSBox-X release published on 2026-10-02 as the provisional SOFIA source baseline, because the user reports **99.6 FPS** in their Quake DOS test with the build downloaded on that date.

Pinned upstream identity:
- Repository: `joncampbell123/dosbox-x`
- Release tag: `dosbox-x-v2026.10.01`
- Tag resolves directly to commit: `b6abbd5980a885f5f310a4088c59a8688d1b116c`
- Release page: `https://github.com/joncampbell123/dosbox-x/releases/tag/dosbox-x-v2026.10.01`
- The release was published 2026-10-02 02:29:52 UTC.
- The tag is a commit object, not an annotated tag.

Important source/release note: the 2026.10.01 release notes explicitly list Sound Blaster AWE32 support and the dynamic-core FPU/MMX routing change. By default, FPU/MMX opcodes are passed to the normal core to support Windows 95 task switching; the notes state that a dosbox.conf option can re-enable dynamic-core handling for DOS games that do not multitask. Therefore, the user's 99.6 FPS result is a benchmark for this release's default behavior, unless their local configuration overrides it. Do not assume that simply switching on dynamic FPU is necessary or beneficial; any such test must be a controlled A/B.

Baseline policy:
1. Use the exact upstream tag/commit above as the initial source snapshot.
2. Preserve upstream history and GPL-2.0 notices; do not rewrite or cherry-pick away history.
3. Keep AWE32/EMU8000 and required ROM workflow, and keep XBRZ enabled.
4. Make IBM-PC/DOS scope reductions only after tracing dependencies; do not remove PC-98-adjacent shared code blindly.
5. Establish a clean baseline build before exclusions. No baseline source build has yet been performed.
6. Preserve the unmodified baseline as a reference and apply one small, documented change at a time.
7. Retest Quake with the user's same scene/settings after every change; retain the user's 99.6 FPS result as a reported reference, not an independently reproduced benchmark.

The October 18 user's 75 FPS measurement remains a separate downloaded binary result. Its exact asset/build provenance and matching test conditions have not yet been confirmed, so the 24.7% difference is an observed user report, not proof that source changes alone caused the regression.


## PC-98 / non-IBM platform audit (2026-11-12)

### Revision coverage

This pass audits the requested upstream snapshot `e2b92bb86382a5444ce42157a37138b459ec98b5` (merge PR #6803, 2026-11-09). The upstream `master` head observed during this pass was `78c022931f0ad9e567bcaaefc155fc428d9bdc7e` (2026-11-12). GitHub compare reports master 28 commits ahead of the candidate and no divergence. The key files that define PC-98 selection and build inclusion have identical blob IDs at both revisions: `configure.ac`, `src/dosbox.cpp`, `include/dosbox.h`, `src/hardware/Makefile.am`, and `src/ints/Makefile.am`. The intervening `src/dos/dos.cpp` change was also checked; its diff does not alter PC-98-specific lines. This is a targeted audit, not a full review of all 28 intervening commits.

### Executive finding

There is **no upstream `--disable-pc98` configure switch** in this snapshot. The default machine is VGA, so PC-98 is not activated in an ordinary default run, but its implementation is still compiled into the program and some PC-98 support assets are still packaged. Runtime defaulting to VGA is not a build-time exclusion.

Existing relevant switches remain separate: `--disable-mt32`, `--disable-sdlnet`, `--disable-monitor`, and `--enable-xbrz`. The configure file contains no PC-98 feature conditional comparable to `C_XBRZ` or `C_MT32`.

### Build-system inclusion points

| Build system / area | Confirmed inclusion | Consequence |
|---|---|---|
| Autotools hardware | `src/hardware/Makefile.am` adds the `snd_pc98/*` include directories unconditionally and lists `pc98.cpp`, `pc98_fm.cpp`, `pc98_artic.cpp`, the `vga_pc98_*.cpp` units, and the `snd_pc98` C sources in `libhardware_a_SOURCES` (lines 1, 16–23). | PC-98 video/audio units are built even when the selected machine is VGA. |
| Autotools interrupts | `src/ints/Makefile.am` lists `pc98_lio.cpp` and `int_dosv.cpp` unconditionally (lines 4–8). | PC-98 LIO and DOS/V code are not separately selectable at build time. |
| Built-in DOS utilities | `src/builtin/Makefile.am` lists `dsxmenu_exe_pc98.cpp` and `cwsdpmi_exe_pc98.cpp`; `include/builtin.h` declares their blobs. | The PC-98-specific utility binaries are compiled into the built-in utility library. Their runtime selection must be traced before removing them. |
| Visual Studio | `vs/dosbox-x.vcxproj` includes PC-98 include directories in its configuration blocks and explicitly compiles the PC-98 hardware/audio sources, `pc98_lio.cpp`, `int_dosv.cpp`, and PC-98 built-in binaries (notably lines 1172, 1181, 1284–1321). | An Autotools-only option would not produce a PC-98-free Windows build. The Visual Studio project must be updated in parallel. |
| Packaged resources | `src/Makefile.am` includes `contrib/fonts/FREECG98.BMP` in `res_DATA` without a PC-98 conditional. | The PC-98 fallback font is shipped even if PC-98 runtime support is unused. |

Source permalinks: [configure.ac](https://github.com/joncampbell123/dosbox-x/blob/e2b92bb86382a5444ce42157a37138b459ec98b5/configure.ac), [hardware Makefile](https://github.com/joncampbell123/dosbox-x/blob/e2b92bb86382a5444ce42157a37138b459ec98b5/src/hardware/Makefile.am), [interrupt Makefile](https://github.com/joncampbell123/dosbox-x/blob/e2b92bb86382a5444ce42157a37138b459ec98b5/src/ints/Makefile.am), [built-in Makefile](https://github.com/joncampbell123/dosbox-x/blob/e2b92bb86382a5444ce42157a37138b459ec98b5/src/builtin/Makefile.am), [Visual Studio project](https://github.com/joncampbell123/dosbox-x/blob/e2b92bb86382a5444ce42157a37138b459ec98b5/vs/dosbox-x.vcxproj), [resource list](https://github.com/joncampbell123/dosbox-x/blob/e2b92bb86382a5444ce42157a37138b459ec98b5/src/Makefile.am).

### Machine selection and shared implementation

- `include/dosbox.h` declares `MCH_PC98`, `IS_PC98_ARCH`, and `PC98_ARCH_CASE` (around lines 129–245). This is the central architecture discriminator.
- `src/dosbox.cpp` defaults to `MCH_VGA`, then parses `machine=pc98`, `pc9801`, and `pc9821` into `MCH_PC98` (around lines 1225–1291). It also contains PC-98 configuration registration and initialization paths.
- `src/hardware/vga.cpp` has a distinct PC-98 entry path (`VGA_OnEnterPC98` and phase 2), configures the PC-98 GDC/VRAM model, and skips several ordinary VGA setup operations when `IS_PC98_ARCH` is true. The PC-98 GDC, EGC, DAC, CRTC, CG, and ARTIC implementations are separate translation units, but they operate through shared VGA state, memory, I/O, timer, and rendering infrastructure.
- The PC-98 path is not confined to those dedicated files. A current-master code-search pass finds `IS_PC98_ARCH` conditionals across many shared modules, including `memory.cpp`, `timer.cpp`, `pic.cpp`, `dma.cpp`, `keyboard.cpp`, `floppy.cpp`, `cmos.cpp`, `ide.cpp`, `vga.cpp`, `bios_keyboard.cpp`, `bios_disk.cpp`, `xms.cpp`, `ems.cpp`, `dos.cpp`, `dos_tables.cpp`, `mouse.cpp`, `shell.cpp`, `sblaster.cpp`, `adlib.cpp`, `gus.cpp`, `mpu401.cpp`, and text/TTF output code. The search returned 68 matching source paths on current master; that count is an index result, not a substitute for a compiler dependency graph.
- The shared modules implement real behavioral differences, not just menu labels: examples include PC-98 PIT clock rates, PIC behavior, A20 control, keyboard IRQ/scancode handling, memory layout, DOS tables, disk services, and video memory mapping. Deleting a dedicated video file alone would leave compile/link references and shared branches behind.

### ROM, BIOS, and font resources

The PC-98 machine does not require the user to provide a separate machine BIOS image just to start emulation. The upstream PC-98 guide says PC-98 mode starts with the executable alone; font resources are optional. In `src/dosbox.cpp`, the initialization path explicitly notes that PC-98 mode has no VGA BIOS.

Keep the following distinctions:

- `FONT.ROM` is optional character/font data, not the machine BIOS. `src/ints/int10.cpp` expects exactly 288,768 bytes when loading this format.
- If `FONT.ROM` is unavailable, the code tries Anex86-compatible bitmap fonts and `FREECG98.BMP`, then has an internal Japanese-font fallback path. Upstream packages `FREECG98.BMP` as a resource. The source itself notes uncertainty about whether the Anex86 font dump may legally be incorporated; do not bundle it casually.
- `SOUND.ROM` is a separate optional 16 KiB PC-98 FM-board sound BIOS image. `src/hardware/pc98_fm.cpp` loads it at physical address `0xCC000` when the PC-98 sound-BIOS option is enabled. If it is absent, the code installs a software INT D2h callback instead. This is not required for ordinary IBM Sound Blaster/AdLib/GUS support.

References: [PC-98 guide](https://github.com/joncampbell123/dosbox-x/wiki/Guide%3APC%E2%80%9098-emulation-in-DOSBox%E2%80%90X), [FONT.ROM and bitmap loading](https://github.com/joncampbell123/dosbox-x/blob/e2b92bb86382a5444ce42157a37138b459ec98b5/src/ints/int10.cpp#L1485-L1652), [SOUND.ROM loading and fallback](https://github.com/joncampbell123/dosbox-x/blob/e2b92bb86382a5444ce42157a37138b459ec98b5/src/hardware/pc98_fm.cpp#L367-L427).

### Video, audio, and licensing boundaries

- **Video:** PC-98 GDC/EGC/DAC/CRTC/CG/ARTIC code is comparatively well grouped into named source files. However, `vga.cpp`, `include/vga.h`, memory access, palette handling, and renderer state are shared. The safe first target is conditional compilation of the dedicated PC-98 translation units after their call sites and declarations are guarded—not deletion of generic VGA files.
- **Audio:** PC-98 FM board emulation is mostly grouped in `pc98_fm.cpp` plus `snd_pc98/*`. It calls shared mixer, PIC/timer, I/O, callback, joystick, and memory services. Standard IBM PC audio units such as Sound Blaster, AdLib/OPL, GUS, MPU-401, and the shared mixer contain PC-98-specific conditionals but are not PC-98-only; retain them.
- **Attribution:** `CREDITS.md` attributes the PC-98 FM board code and GDC/LIO drawing code to Neko Project II under BSD 3-clause, and JEGA/DOS/V code to nanshiki/Wengier under GPLv2+. Any later pruning must preserve notices for retained code and respect each component's license.

### Other non-IBM or regional modes

These should be treated as separate feature decisions, not folded into a single PC-98 deletion:

- **JEGA / AX:** `machine=jega` maps to `MCH_EGA` with a JEGA flag; it is not a separate machine enum. It shares EGA, INT 10h, font, and DBCS code.
- **DOS/V and Toshiba J-3100:** configured through the `[dosv]` section and implemented in shared DOS/INT 10h/DBCS paths; `int_dosv.cpp` is in the common interrupt source list. Do not remove generic DBCS/codepage/TTF support as a side effect.
- **FM Towns:** `MCH_FM_TOWNS` is marked `STUB` in `include/dosbox.h`; `machine=fm_towns` currently maps through VGA plus a special flag. It is not an isolated full machine implementation.
- **Tandy/PCjr, Amstrad, Olivetti M24/AT&T 6300, MCGA, Hercules, and IBM 3270 PC:** these are PC-compatible or IBM-family machine/video variants, not candidates for blanket removal under SOFIA's stated IBM-PC-compatible scope. Keep them unless a narrower policy explicitly excludes a particular profile.

### Safe decisions and remaining work

1. **Safe now:** leave all PC-98 source files in place; keep the default IBM-compatible VGA machine path; retain XBRZ and standard IBM PC audio; continue using the already verified MT-32 and SDL networking switches. This changes no upstream code and risks no compatibility regression.
2. **Do not claim PC-98 is excluded today:** it is inactive by default, but still compiled and packaged. There is no upstream PC-98 build switch.
3. **Future true build-time exclusion:** add a SOFIA-specific `--disable-pc98` / `C_PC98` path only after guarding every PC-98 reference in shared files, splitting the Autotools and Visual Studio source lists, handling PC-98 built-in utility selectors and font resources, and checking all link dependencies. Keep DOS/V/JEGA as separately selectable features (for example, `C_DOSV` / `C_JEGA`) rather than treating them as synonyms for PC-98.
4. **Do not delete shared files** such as `dosbox.cpp`, `dosbox.h`, `vga.cpp`, `memory.cpp`, `timer.cpp`, `pic.cpp`, `keyboard.cpp`, `bios_keyboard.cpp`, `bios_disk.cpp`, `xms.cpp`, `ems.cpp`, `dos.cpp`, `dos_tables.cpp`, `int10.cpp`, `sblaster.cpp`, `adlib.cpp`, `gus.cpp`, or `mpu401.cpp` based on PC-98 references alone.
5. **Next step:** establish the exact source tree for the chosen SOFIA baseline, then make a dependency inventory for one feature boundary at a time—first PC-98 GDC/video, then PC-98 FM/audio, then shared architecture guards and built-in resources. Build the unmodified baseline before enabling any exclusion, and test DOS boot, VGA modes, storage, input, Sound Blaster/AdLib/GUS, and Quake DOS with XBRZ after each step.

No source files were deleted, no PC-98 switch was added, and no build or runtime test was executed in this audit.
