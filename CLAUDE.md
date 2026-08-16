# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single Dockerfile (`Dockerfile-single`) that builds an Ubuntu 24.04 container image with SPDK (Storage Performance Development Kit) and its DPDK (Data Plane Development Kit) submodule cloned and ready to compile. The goal (per `README.md`) is a self-contained image that can be transported into airgapped environments — i.e. all build tooling and source is baked into the image rather than fetched at container-run time.

## Build

```bash
docker build -f Dockerfile-single -t spdk-dpdk-ubuntu .
```

There is no application code, test suite, or lint config in this repo — it's infrastructure-as-a-Dockerfile. "Testing" a change means rebuilding the image and, if needed, running it (`docker run -it spdk-dpdk-ubuntu`) to confirm the toolchain and SPDK checkout behave as expected.

## Structure and build flow

`Dockerfile-single` is a single-stage build that runs, in order:

1. Installs build toolchain and runtime libs via `apt-get` (meson/ninja/nasm for DPDK, libnuma/libaio/libfuse3/libssl/uuid-dev for SPDK, plus `python3-*` bindings SPDK's build scripts expect) and installs `uv` via pip.
2. Copies `Dockerfile-single`, `Dockerfile`, and `README.md` into `/workspace` inside the image — this embeds the build recipe itself in the image so it can be inspected/rebuilt from within an airgapped container. **Note:** only `Dockerfile-single` currently exists in this repo; the `COPY Dockerfile /workspace/Dockerfile` line will fail the build until a file literally named `Dockerfile` is added alongside it.
3. Clones SPDK at tag `v26.05` (shallow, depth 1) into `/workspace/spdk` and initializes its submodules (which includes the DPDK and isa-l submodules).
4. Runs SPDK's own `./scripts/pkgdep.sh --no-check-hugepages` to let SPDK's dependency resolver fill in anything the manual apt-get list missed.
5. Sets `PKG_CONFIG_PATH`/`CFLAGS`/`LDFLAGS` so a subsequent SPDK build can find the vendored `isa-l` submodule.
6. The actual SPDK `configure && make` and DPDK `meson`/`ninja` build steps are present but **commented out** — the image currently ships source + toolchain only, not compiled binaries. If uncommenting these, note the DPDK meson step is deliberately set to `-Dcpu_instruction_set=corei7` (a conservative/generic instruction set) rather than the default `native`, specifically to avoid SIGILL crashes when the built image runs on different CPU hardware than it was built on.
7. `PATH`/`LD_LIBRARY_PATH` are pre-wired to `/workspace/spdk_package/{bin,lib}` in anticipation of an `make install` step landing binaries there.

When modifying this Dockerfile, preserve the airgap intent: avoid adding steps that require network access beyond the initial `apt-get`/`git clone`/`pip install` (e.g. don't add a `pip install` that isn't `--break-system-packages`-safe or that assumes PyPI access at container-run time).
