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
