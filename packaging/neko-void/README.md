# Proton Mail Bridge for Neko-Void

This directory contains the xbps-src template for Proton Mail Bridge, ready to be
submitted to [Neko-Void-Linux/neko-pkgs](https://github.com/Neko-Void-Linux/neko-pkgs).
Neko-Void is a Void Linux respin, so the template follows Void packaging rules
(`xbps-src`, glibc, x86_64) and works on plain Void too.

## What gets built

* `bridge` — the Bridge daemon and CLI, built directly from `cmd/Desktop-Bridge`.
  Proton's signed launcher is intentionally not packaged: it verifies Proton's
  signature and is only usable by the official distribution.
* `bridge-gui` — the Qt 6 desktop application, linked against the system Qt,
  gRPC, Protobuf, and Sentry instead of the bundled vcpkg copies.

Both binaries are installed in `/usr/lib/protonmail/bridge` (the GUI locates the
daemon next to itself). `/usr/bin/protonmail-bridge` starts the GUI and
`/usr/bin/protonmail-bridge-core` starts the daemon/CLI.

## Build and install

```sh
git clone --depth 1 https://github.com/void-linux/void-packages
cd void-packages

mkdir -p srcpkgs/protonmail-bridge
cp -r /path/to/proton-bridge/packaging/neko-void/protonmail-bridge/* \
      srcpkgs/protonmail-bridge/

./xbps-src pkg protonmail-bridge
sudo xbps-install --repository=hostdir/binpkgs protonmail-bridge
```

Alternatively copy the same directory into a checkout of `neko-pkgs` and build it
with [vouru](https://github.com/Neko-Void-Linux/vouru).

## Runtime requirements

Bridge needs a freedesktop secret service to store the account credentials:

```sh
sudo xbps-install gnome-keyring   # or: pass
```

Bridge enables its own XDG autostart entry by default, so no runit service is
needed on a desktop installation. The CLI can be run in the foreground with
`protonmail-bridge-core --cli` or `--noninteractive`.

## Notes for the package maintainer

* `files/system-deps.patch` adds the `BRIDGE_USE_SYSTEM_DEPS` CMake option.
  `post_patch()` skips it once that option is part of a Proton Mail Bridge
  release, so the template keeps working after the upstream change lands.
* `files/protobuf-include.patch` passes Protobuf's include directory to
  `protoc`. System Protobuf installations need it to resolve imports such as
  `google/protobuf/empty.proto`; vcpkg hid the missing argument. It is skipped
  the same way once the fix is released upstream.
* Replace the `maintainer=` line in `template` with the person submitting the
  package to `neko-pkgs`.
* Bump `version` and `checksum` together. The checksum is the SHA-256 of
  `https://github.com/ProtonMail/proton-bridge/archive/refs/tags/v<version>.tar.gz`.

## Verification

The template was exercised on a real Void x86_64 glibc userland (Void rootfs
with the declared package set installed through xbps; Go 1.27.4, Qt 6.11.2,
gRPC 1.83.0, Protobuf 35.1, sentry-native 0.17.6, libfido2 1.17.0, libcbor
0.14.0, SQLite 3.54.1):

* `post_patch`, `do_build` and `do_install` from this template were run against
  the pristine `v3.27.2` release tarball with the checksum above.
* `bridge --help` and `bridge --version` (3.27.2) work, and the daemon links the
  system `libfido2`, `libcbor`, `libsqlite3` and `libsecret`.
* `bridge-gui` links the system Qt 6.11.2, gRPC and Sentry. It starts with
  `QT_QPA_PLATFORM=offscreen`, logs `Using Qt 6.11.2`, and finds the daemon at
  `/usr/lib/protonmail/bridge/bridge`.
* The installed wrappers work: `protonmail-bridge-core --version` prints
  `Proton Mail Bridge 3.27.2`.

Not covered: a full `./xbps-src pkg` run (this environment cannot create the
root-owned xbps-src masterdir) and visual GUI testing (no display server). The
GUI was not clicked through.
