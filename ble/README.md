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
| `ble/macos/cn1-ble-helper` | macOS `x86_64` **and** `arm64` | native builds joined with `lipo` |
| `ble/linux/x64/cn1-ble-helper` | Linux `x86_64`, glibc 2.36+ | `rust:1-bookworm` container |
| `ble/linux/arm64/cn1-ble-helper` | Linux `aarch64`, glibc 2.36+ | `rust:1-bookworm` container |
| `ble/windows/x64/cn1-ble-helper.exe` | Windows `x86_64` | cross-compiled, `x86_64-pc-windows-msvc` |
| `ble/windows/arm64/cn1-ble-helper.exe` | Windows `arm64` | cross-compiled, `aarch64-pc-windows-msvc` |

macOS gets a single universal Mach-O binary because the format supports it;
ELF and PE have no fat-binary equivalent, so Linux and Windows ship one
binary per architecture and `NativeBleBackend.helperResourcePath()` resolves
`os.arch` (`amd64`/`x86_64` → `x64`, `aarch64`/`arm64` → `arm64`) into the
directory name. Architectures with no bundled binary — 32-bit x86, 32-bit
ARM — fall through to the `PATH` lookup, so a self-built helper still works
there.

The Windows binaries link the CRT statically and the Linux ones need only
`libdbus-1.so.3` (present wherever BlueZ is), so none of them require a
redistributable.

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

# Linux, per arch (glibc; needs the D-Bus headers for BlueZ).
# --platform picks the arch; it is emulated unless it matches the host.
for arch in amd64 arm64; do
  docker run --rm --platform linux/$arch -v "$PWD":/src -w /src rust:1-bookworm \
      bash -c "apt-get update && apt-get install -y pkg-config libdbus-1-dev && \
               cargo build --release"
done

# Windows, per arch, cross-compiled with cargo-xwin (cargo install cargo-xwin).
# crt-static keeps the exe free of a VCRUNTIME140.dll dependency, which is
# NOT present on a clean Windows install.
export RUSTFLAGS="-C target-feature=+crt-static"
cargo xwin build --release --target x86_64-pc-windows-msvc
cargo xwin build --release --target aarch64-pc-windows-msvc
```

After building, confirm the Windows binaries import only system DLLs
(`KERNEL32`, `ntdll`, `ole32`, `oleaut32`, `bcryptprimitives`,
`api-ms-win-core-winrt-*`) and nothing like `VCRUNTIME140.dll`:

```bash
llvm-objdump -p cn1-ble-helper.exe | grep 'DLL Name' | sort -u
```

## Validation status

| Binary | Verified |
|---|---|
| macOS (universal) | Ran against a real radio: capabilities handshake, `poweredOn`, live scan results; both slices present per `lipo -info`. |
| Linux x64 | Executed on `debian:bookworm-slim` (x86_64): emits the capabilities handshake, and without a D-Bus/BlueZ adapter reports "no usable adapter … unsupported" and exits cleanly, as designed. Not yet exercised against a real Linux radio. |
| Linux arm64 | Same, executed on `debian:bookworm-slim` (aarch64). |
| Windows x64 / arm64 | Verified as PE32+ console executables of the expected architecture, importing only system DLLs (WinRT among them) — but **not executed**, being cross-builds. Run them on real Windows hardware before relying on them. |

A missing or broken helper degrades gracefully: the JavaSE port reports the
native backend as unavailable and the simulator's virtual Bluetooth stack
keeps working.

Keep the executable bit set on the Unix binaries (`chmod +x`); the Java side
re-applies it after extracting to a temp file, but a correct mode here keeps
direct use from a checkout working.
