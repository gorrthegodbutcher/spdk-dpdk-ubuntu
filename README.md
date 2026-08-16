A docker container project that is self-contained and can be transported to airgapped environments.

On the host, before running the container, you need to allocate hugepages
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

run or exec into the running container
docker run --rm --privileged -v /dev:/dev -v /dev/shm:/dev/shm -v /dev/hugepages:/dev/hugepages -it <container image> /bin/bash


### Building on the target hardware

The image ships SPDK/DPDK source + toolchain only - it does not bake in
precompiled binaries. Compile everything inside the container, on the actual
target machine, so the code is built for that machine's real CPU instead of
whatever CPU built the image.

If you always build fresh, inside a container running directly on the box
that will also run the result, leave `--target-arch` unset. The default is
`native` - detect whatever CPU is physically running the compiler and use
every instruction it supports. That's the best choice when build and run
are the same machine: full performance, and no possibility of hitting an
unsupported instruction. Docker doesn't get in the way of this - a
container shares the host kernel directly (it's not a VM), so `native`
correctly sees the real physical CPU.

```
cd /workspace/spdk
./configure --prefix=/workspace/spdk_package --disable-tests --disable-unit-tests
make -j$(nproc) && make install
```

Only override `--target-arch` if build and run can end up on *different*
machines - e.g. compiling in WSL2 and running the result on real hardware
(what originally broke this image with "invalid CPU" / SIGILL), or building
once and copying the result to other boxes in a fleet without rebuilding on
each one. In that case, pick an explicit target instead of `native`:

- HP ProLiant DL360 Gen9 (Xeon E5-26xx v3/v4, Haswell/Broadwell-EP, no
  AVX-512): use `haswell` - covers both v3 and v4 safely with real AVX2
  performance. Pass `--target-arch=haswell` to `./configure`.
- Unknown/mixed fleet, or if `haswell` still SIGILLs: fall back to `corei7`,
  DPDK's lowest-common-denominator x86_64 baseline.

This also builds and installs the small subset of DPDK libraries SPDK
itself needs (vhost, cryptodev, dmadev, ...). It's enough for SPDK's own
binaries and for the simpler DPDK examples (helloworld, ethtool, l2fwd,
vhost, ...), but not for examples needing DPDK libraries SPDK doesn't build
(rte_lpm, rte_acl, rte_sched, rte_eventdev, ...). For full example coverage,
also build DPDK standalone (leave `-Dcpu_instruction_set` unset for the same
build-equals-run reasoning as above, or match whatever `--target-arch` you
used for SPDK if build and run differ):

```
cd /workspace/spdk/dpdk
meson setup /workspace/dpdk_build
ninja -C /workspace/dpdk_build
ninja -C /workspace/dpdk_build install
ldconfig
```

`PKG_CONFIG_PATH` is already set up to prefer this full install over SPDK's
partial one whenever both are present, so nothing else needs to change to
pick it up.

### To build examples in the DPDK directory

```
cd /workspace/spdk/dpdk/examples/<name>
make
```

Each of these builds standalone via `pkg-config libdpdk` - no dependency on
the SPDK build above beyond the DPDK install itself.

### SPDK's own examples (`/workspace/spdk/examples/`)

Not the same thing as the DPDK examples above, and not built the same way.
These do not build standalone - they depend on `mk/config.mk`, which only
exists after `./configure` has been run at the SPDK root (see "Building on
the target hardware"). `cd`-ing straight into one of these and running
`make` before that gives:

```
mk/config.mk: file not found. Please run configure before make.
```

In practice this whole category needs no separate step: `make` at the SPDK
root (from the build above) already builds every SPDK example
automatically, landing binaries under `/workspace/spdk/build/examples/` -
not next to the source, unlike the DPDK examples. `cd`-ing into a specific
example's source directory and running `make` there only makes sense
afterward, to rebuild just that one example.
