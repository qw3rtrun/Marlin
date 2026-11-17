# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Start here

- `AGENTS.md` (repo root) is the canonical orientation doc: architecture, G-code pipeline, singletons, PR rules. Read it rather than re-deriving the big picture.
- Before touching a HAL, read `Marlin/src/HAL/<FAMILY>/AGENTS.md`.
- `buildroot/skills/<name>/SKILL.md` holds verified step-by-step procedures: build-test, config-options, code-style, pr-workflow, solving-bugs, extui-backends, unit-tests, codebase-navigation, hal-work, fork-harvesting. Follow the matching skill instead of improvising.

## Commands

Run everything from the repo root. PlatformIO (`pio`) must be on PATH. The `buildroot/bin/*` scripts and the `Makefile` are POSIX shell, so use the Bash tool (Git Bash) on this Windows machine, not PowerShell.

- Build the configured board: `buildroot/bin/mftest -a` (or `make marlin`). List the envs for the board with `buildroot/bin/mfenvs`. Build one env with `pio run -e <env>`.
- Windows simulator: `pio run -e simulator_windows` (needs MSYS2 mingw64 with SDL2/SDL2_net/glm; see the comments in `ini/native.ini`). Linux/macOS: `simulator_linux_debug` / `simulator_macos_debug`.
- CI build test, one config: `buildroot/bin/mftest -t <env> -n <index> -y` or `make tests-single-local TEST_TARGET=<env> ONLY_TEST=<index>`. Test configs are `buildroot/tests/<env>/config-NN.ini`.
- Unit tests (native `linux_native_test` env): all with `make unit-test-all-local`; one config with `make unit-test-single-local UNIT_TEST_CONFIG=<name>`, where `<name>` is the `test/NNN-<name>.ini` name without the number. Test sources are in `Marlin/tests/<area>/test_*.cpp` and the configs are in `test/*.ini`.
- Validation: `make validate-lines`, `make validate-pins -j`, `make validate-boards`, `make validate-conditionals`.
- Config edits from the CLI: `buildroot/bin/opt_enable`, `opt_disable`, `opt_set`, `opt_find`.

**Warning:** The test runners and `buildroot/bin/restore_configs` run `git checkout` on `Configuration*.h`, and in CI mode they also run `git reset --hard`. Commit or stash other work before you run them.

## Configuration lives in ini files, not in the headers

`Marlin/Configuration.h` and `Configuration_adv.h` are kept identical to upstream `bugfix-2.1.x`. **Never edit them by hand, and never commit them in a modified state.** Every machine setting goes in an ini profile under `config/examples/<machine>/`.

Before every build, `buildroot/share/PlatformIO/scripts/configuration.py` applies `Marlin/config.ini`. The active `ini_use_config` line looks like this:

`[reset], base +basic +host +debug @../config/examples/simulation/config.ini`

1. `[reset]` (added in this fork) runs `git checkout HEAD` on both headers, so each build starts from pristine headers and no settings leak in from earlier builds or other profiles. It also drops the timestamp from the `// Added by config.ini` comments, and it keeps the headers' file times when the result is unchanged, so repeat builds stay incremental.
2. `base +basic +host +debug @<file>` loads that profile (path relative to `Marlin/`) and applies the named `[config:<name>]` sections in order. This fork's `apply_sections()` accepts several `+`-joined names, where upstream takes one. A name with no matching section prints `Section [...] not found`.
3. Options that don't exist in the headers get inserted near the top of `Configuration.h` (this is how the ET4000+ `button4_*`/`button5_*` options are defined).

To switch machines, uncomment a different `ini_use_config` line in `Marlin/config.ini`. Each profile's `[config:base]` comment has its line. To change a setting, edit the profile. After a build, `git status` shows the headers as modified. That's expected, so don't commit them.

Profiles: `simulation` (BOARD_SIMULATED), `NoviS3D/config.ini` (MKS Robin Lite), `NoviS3D/nano_config.ini` (MKS Robin Nano V3.1), `EasyThreeD/ET4000PLUS` (EASYTHREED_UI, custom buttons, backlash).

## Branches

- `bugfix-2.1.x-cfg` is this fork's default branch. It holds the fork-local machine and simulator configuration, and fork work branches from and merges back into it.
- `bugfix-2.1.x` mirrors upstream (MarlinFirmware/Marlin). Branch upstream PRs from it, and keep the fork's machine configs, `config.ini` choices and `ini/native.ini` Windows tweaks out of them.

## Code conventions that bite

- Code must build on 8-bit AVR and on 32-bit targets. Gate optional code with `ENABLED()` / `DISABLED()` / `ANY()` / `HAS_*`, and add validation to `Marlin/src/inc/SanityCheck.h`.
- Config and conditional headers are applied in the strict order set by `Marlin/src/inc/MarlinConfig.h` (the `MarlinConfigPre-N-*.h` / `Conditionals-N-*.h` chain: `Configuration.h` is pulled in at stage 1-axes and `Configuration_adv.h` at 4-adv, then `5-post`, `6-type`, and finally `SanityCheck.h`). A derived flag is only visible after the header that defines it.
- Use 2-space indents, and indent preprocessor nesting like brace nesting. Don't run `format_code`/`uncrust`/clang-format on existing files. The untracked `.clang-format` is not project style.
- Commits are imperative, ≤72 chars, and start with a gitmoji (🐛 ✨ 🔧 🩹 ⚡️ 🧑‍💻 🚸).
