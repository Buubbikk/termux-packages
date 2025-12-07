## Purpose

This file gives concise, actionable guidance to AI coding agents working on the `termux-packages` repository — how the build system is organized, important conventions, and exact commands and file locations to use when making changes.

## Big picture

- **Repo role:** collection of package build scripts and build toolchain wrappers used to produce Termux packages (Debian / Pacman) for multiple libraries (`bionic` vs `glibc`).
- **Major components:**
  - `packages/` — package directories, each with a `build.sh` declaring `TERMUX_PKG_*` variables and optional hook functions (`termux_step_*`).
  - `scripts/` — the reusable build system: `scripts/build/*` contains build steps invoked by `build-package.sh`.
  - Top-level helpers: `build-package.sh`, `build-all.sh`, `scripts/run-docker.sh`.
  - `repo.json` — repo-wide settings (package format, urls).

## Key developer workflows (concrete commands)

- Start an isolated builder (recommended):

  ```sh
  ./scripts/run-docker.sh
  ```

- Build a single package (example `bash`):

  ```sh
  ./build-package.sh -i bash
  ```

- Build all packages (uses build order):

  ```sh
  ./build-all.sh -a aarch64
  ```

- Common options:
  - `-i` (build-package): download/extract dependencies instead of building them
  - `-a ARCH` (build scripts): `aarch64`, `arm`, `i686`, `x86_64` or `all`
  - `-o DIR` to set output directory
  - `--format` and `--library` to control package format and library (debian/pacman, bionic/glibc)

## Package format and conventions

- Package metadata lives in `packages/<pkg>/build.sh`. Common fields: `TERMUX_PKG_NAME`, `TERMUX_PKG_VERSION`, `TERMUX_PKG_SRCURL`, `TERMUX_PKG_SHA256`, `TERMUX_PKG_DEPENDS`, `TERMUX_PKG_MAINTAINER`.
- Hooks and overrides: packages implement or override lifecycle functions named `termux_step_get_source`, `termux_step_pre_configure`, `termux_step_post_make_install`, etc. The core `build-package.sh` sources `scripts/build/*` steps and then calls these hooks — use hooks, do not edit core flow unless necessary.
- Patches: package-specific `.patch` files are applied from `packages/<pkg>/*.patch`.
- Avoid `sudo` in package scripts — `build-package.sh` traps and disallows `sudo` usage.

## Where outputs and logs appear

- Built artifacts: `output/` (or `-o` specified) — e.g. `output/bash_5.2.26-2_aarch64.deb`.
- Build-all logs/status: `$TERMUX_TOPDIR/_buildall-<arch>/ALL.out` and per-package `<PKG>.out` in the same directory. Inspect these when debugging CI/local failures.

## Code patterns and examples (do this, not that)

- Look at `packages/cmake/build.sh` for variable patterns and an example `termux_pkg_auto_update()` implementation.
- Use `TERMUX_PKG_*` variables for metadata and `termux_pkg_auto_update()` when implementing auto-update logic.
- When changing how packages build, prefer adding or overriding `termux_step_*` hooks inside the package's `build.sh` rather than changing `scripts/build/*` globally.

## Integration points & external dependencies

- Docker image used by `scripts/run-docker.sh`: `ghcr.io/termux-play-store/package-builder` (configured in script). Adjust with `TERMUX_BUILDER_IMAGE_NAME` env var.
- `repo.json` controls package format and repository metadata — ensure changes remain compatible with `build-package.sh` validations.
- Some packages call external toolchains (NDK/toolchain scripts under `scripts/build/toolchain/`) — be careful when updating toolchain setup logic; tests often require cross-compiling behaviors.

## Quick debugging checklist for CI failures

- Reproduce locally with `./scripts/run-docker.sh` and then `./build-package.sh` with the same args as CI.
- Inspect `$TERMUX_TOPDIR/_buildall-<arch>/*.out` and `output/` for the failing package.
- Check for missing PGP keys: build scripts import the repo keyring from `packages/termux-keyring/termux-packages.gpg` when `-i` or `glibc` downloads are used.

## Where to look for more context

- `scripts/build/` — build step implementations and lifecycle order.
- `scripts/run-docker.sh` and `scripts/utils/docker/docker.sh` — container entry and trap handling.
- `repo.json` — package format and repo metadata.
- `packages/` — canonical examples of `build.sh` structure and patch usage.

If any section is unclear or you'd like more specific examples (e.g. merge examples for an existing `.github/copilot-instructions.md`), tell me which area to expand.
