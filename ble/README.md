# cn1-ble-helper binaries

Prebuilt `cn1-ble-helper` executables for the Codename One JavaSE port's
**native Bluetooth backend**. The simulator ships two interchangeable
backends: a scriptable virtual stack (the default) and this helper, which
talks to the host machine's real Bluetooth radio so the simulator can scan
and connect to actual devices.

The helper is a small Rust program built on
[btleplug](https://github.com/deviceplug/btleplug) (CoreBluetooth on macOS,
BlueZ/D-Bus on Linux, WinRT on Windows). It speaks line-delimited JSON over
stdin/stdout; the protocol is documented in `PROTOCOL.md` next to the
source.

## Layout

| Path | Host | Built as |
|---|---|---|
| `ble/macos/cn1-ble-helper` | macOS, universal (`x86_64` + `arm64`) | native + `lipo` |
| `ble/linux/cn1-ble-helper` | Linux `x86_64` (glibc 2.36+) | `rust:1-bookworm` container |
| `ble/windows/cn1-ble-helper.exe` | Windows `x86_64` | cross-compiled, `x86_64-pc-windows-gnu` |

`maven/javase/pom.xml` maps this directory into the JavaSE jar at
`com/codename1/impl/javase/bluetooth/native/`, where
`NativeBleBackend.helperResourcePath()` loads it. The mapping is optional:
a checkout without `ble/` still builds, and the native backend then simply
reports itself unavailable (the simulator's virtual stack is unaffected).

At runtime users can override the binary with
`-Dcn1.bluetooth.helperPath=<path>`; a `PATH` lookup is the last resort.

## Source of truth

The source lives in the CodenameOne repository at
`Ports/JavaSE/native/cn1-ble-helper/` — these binaries are built from it and
must be refreshed whenever it changes. `Cargo.lock` is committed there, so
rebuilds are reproducible.

## Rebuilding

Maintainers can build the helper for the current host without touching this
repository:

```bash
cd maven && mvn package -pl javase -Pbuild-ble-helper
```

To refresh the binaries here:

```bash
cd Ports/JavaSE/native/cn1-ble-helper

# macOS universal
rustup target add aarch64-apple-darwin x86_64-apple-darwin
cargo build --release --target aarch64-apple-darwin
cargo build --release --target x86_64-apple-darwin
lipo -create -output cn1-ble-helper \
    target/aarch64-apple-darwin/release/cn1-ble-helper \
    target/x86_64-apple-darwin/release/cn1-ble-helper

# Linux x86_64 (glibc; needs the D-Bus headers for BlueZ)
docker run --rm --platform linux/amd64 -v "$PWD":/src -w /src rust:1-bookworm \
    bash -c "apt-get update && apt-get install -y pkg-config libdbus-1-dev && \
             cargo build --release"

# Windows x86_64, cross-compiled from macOS/Linux (brew install mingw-w64)
rustup target add x86_64-pc-windows-gnu
cargo build --release --target x86_64-pc-windows-gnu

# ...or natively on a Windows host with the MSVC toolchain
cargo build --release --target x86_64-pc-windows-msvc
```

## Validation status

| Binary | Verified |
|---|---|
| macOS | Ran against a real radio: capabilities handshake, `poweredOn`, live scan results. |
| Linux | Runs on `debian:bookworm-slim`; without a D-Bus/BlueZ adapter it reports "no usable adapter … unsupported" and exits cleanly, as designed. Not yet exercised against a real Linux radio. |
| Windows | Built and verified as a PE32+ x86-64 console executable, but **not executed** — it is a mingw (`-gnu`) cross-build of btleplug's WinRT bindings and should be exercised on a Windows box before being relied on. |

A missing or broken helper degrades gracefully: the JavaSE port reports the
native backend as unavailable and the simulator's virtual Bluetooth stack
keeps working.

Keep the executable bit set on the Unix binaries (`chmod +x`); the Java side
re-applies it after extracting to a temp file, but a correct mode here keeps
direct use from a checkout working.
