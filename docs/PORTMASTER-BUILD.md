# box64 for PortMaster ports (branch `portmaster`)

This is [box64](https://github.com/ptitSeb/box64) 0.4.4 with one small change, built as a single portable binary
for ARMv8-A handhelds running PortMaster firmwares (ROCKNIX, Knulli, muOS, ArkOS, AmberELEC and others). It is
the box64 shipped with the Ittle Dew port.

## The change

`src/librarian/library.c`, `initWrappedLib()`: when one of a wrapped library's own "needed" native helper
libraries cannot be loaded, box64 no longer prints two errors at the always-on log level and no longer returns
before the wrapped library's link-map setup. The wrapped library itself has loaded; only calls into the missing
helper would fail, and the messages are now at `BOX64_LOG=1` level.

Why: a port can run a Unity 4 player with a libX11 stand-in instead of an X server (no Weston, no Xwayland). The
libX11 wrapper then wants `libxcb.so.1`, the libXcursor wrapper `libXfixes.so.3` and `libXrender.so.1`, and none
of them exist on the device; nothing calls them.

## Build

Cross-compiled on Ubuntu 24.04 (`gcc-aarch64-linux-gnu`, `cmake`, `symlinks`) for any ARMv8-A core (no ARMv8.2
instructions, so it runs on Cortex-A53/A35 devices too), linked against a Debian 10 sysroot so that it needs only
glibc 2.28. `-DBAD_SIGNAL=ON` is required on Rockchip kernels (Mono's JIT crashes without it).

1. Sysroot: the Debian 10 arm64 packages `libc6`, `libc6-dev` (`pool/main/g/glibc/`) and `linux-libc-dev`
   (`pool/main/l/linux/`) from archive.debian.org, extracted into one folder.

```
SR=$HOME/sysroot-buster/root; mkdir -p $SR debs; cd debs
wget http://archive.debian.org/debian/pool/main/g/glibc/libc6_2.28-10+deb10u1_arm64.deb
wget http://archive.debian.org/debian/pool/main/g/glibc/libc6-dev_2.28-10+deb10u1_arm64.deb
wget http://archive.debian.org/debian/pool/main/l/linux/linux-libc-dev_4.19.249-2_arm64.deb
for p in *.deb; do dpkg-deb -x $p $SR; done
symlinks -cr $SR
```

2. box64:

```
git clone --branch portmaster --depth 1 https://github.com/Knifethrower/box64-portmaster.git && cd box64-portmaster && mkdir build && cd build
SR=$HOME/sysroot-buster/root
FLAGS="-nostdinc -isystem $(aarch64-linux-gnu-gcc -print-file-name=include) -isystem $SR/usr/include/aarch64-linux-gnu -isystem $SR/usr/include --sysroot=$SR -march=armv8-a -mtune=cortex-a55"
cmake .. -DARM64=1 -DARM_DYNAREC=ON -DBAD_SIGNAL=ON -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DCMAKE_C_COMPILER=aarch64-linux-gnu-gcc -DCMAKE_ASM_COMPILER=aarch64-linux-gnu-gcc \
  -DCMAKE_C_FLAGS="$FLAGS" -DCMAKE_ASM_FLAGS="$FLAGS" \
  -DCMAKE_EXE_LINKER_FLAGS="--sysroot=$SR -B$SR/usr/lib/aarch64-linux-gnu -L$SR/lib/aarch64-linux-gnu -L$SR/usr/lib/aarch64-linux-gnu -Wl,-rpath-link,$SR/lib/aarch64-linux-gnu"
make -j$(nproc)
aarch64-linux-gnu-strip box64
objdump -T box64 | grep -o 'GLIBC_[0-9.]*' | sort -V | tail -1   # GLIBC_2.28
```

The result is one 25 MB binary that a port ships as `<port>/box64`.

## Licence

box64 is MIT licensed (see `LICENSE`); this change is offered under the same terms.
