# libcn1ble shared libraries

Prebuilt `libcn1ble` shared libraries for the Codename One JavaSE port's
**native Bluetooth backend**. The simulator ships two interchangeable
backends: a scriptable virtual stack (the default) and this library, which
talks to the host machine's real Bluetooth radio so the simulator can scan
and connect to actual devices.

The library is a small Rust `cdylib` built on
[btleplug](https://github.com/deviceplug/btleplug) (CoreBluetooth on macOS,
BlueZ/D-Bus on Linux, WinRT on Windows). It is loaded in-process and exposes
a JNI ABI (`JniBleBridge`) for the JavaSE port and a C ABI for ParparVM; the
event protocol is documented in `PROTOCOL.md` next to the source. It replaces
the earlier `cn1-ble-helper` stdin/stdout subprocess executables.

## Layout

| Path | Host | Built as |
|---|---|---|
| `ble/macos/libcn1ble.dylib` | macOS `x86_64` **and** `arm64` | native builds joined with `lipo` |
| `ble/linux/x64/libcn1ble.so` | Linux `x86_64`, glibc 2.36+ | `rust:1-bookworm` container |
| `ble/linux/arm64/libcn1ble.so` | Linux `aarch64`, glibc 2.36+ | `rust:1-bookworm` container |
| `ble/windows/x64/cn1ble.dll` | Windows `x86_64` | cross-compiled, `x86_64-pc-windows-gnu` |
| `ble/windows/arm64/cn1ble.dll` | Windows `arm64` | cross-compiled, `aarch64-pc-windows-gnullvm` |

macOS gets a single universal Mach-O library because the format supports it;
ELF and PE have no fat-binary equivalent, so Linux and Windows ship one
library per architecture and `BleLibraryResolver.libraryResourcePath()`
resolves `os.arch` (`amd64`/`x86_64` → `x64`, `aarch64`/`arm64` → `arm64`)
into the directory name. Architectures with no bundled library — 32-bit x86,
32-bit ARM — report the native backend as unavailable and fall back to the
virtual stack.

The Linux libraries need only `libdbus-1.so.3` (present wherever BlueZ is),
so none of them require a redistributable.

`maven/javase/pom.xml` maps this directory into the JavaSE jar at
`com/codename1/impl/javase/bluetooth/native/`, where
`BleLibraryResolver` extracts and `System.load()`s it. The mapping is
optional: a checkout without `ble/` still builds, and the native backend
then simply reports itself unavailable (the simulator's virtual stack is
unaffected).

At runtime users can override the library with
`-Dcn1.bluetooth.libraryPath=<path>`.

## Source of truth

The source lives in the CodenameOne repository at
`Ports/JavaSE/native/cn1-ble-helper/` — these libraries are built from it and
must be refreshed whenever it changes. `Cargo.lock` is committed there, so
rebuilds are reproducible.

## Rebuilding

To refresh the libraries here:

```bash
cd Ports/JavaSE/native/cn1-ble-helper

# macOS universal
rustup target add aarch64-apple-darwin x86_64-apple-darwin
cargo build --release --target aarch64-apple-darwin
cargo build --release --target x86_64-apple-darwin
lipo -create -output libcn1ble.dylib \
    target/aarch64-apple-darwin/release/libcn1ble.dylib \
    target/x86_64-apple-darwin/release/libcn1ble.dylib

# Linux, per arch (glibc; needs the D-Bus headers for BlueZ).
# --platform picks the arch; it is emulated unless it matches the host.
# (docker or podman)
for arch in amd64 arm64; do
  docker run --rm --platform linux/$arch -v "$PWD":/src -w /src rust:1-bookworm \
      bash -c "apt-get update && apt-get install -y pkg-config libdbus-1-dev && \
               cargo build --release"   # -> target/release/libcn1ble.so
done

# Windows x64, cross-compiled with the mingw-w64 GCC toolchain.
rustup target add x86_64-pc-windows-gnu       # + brew install mingw-w64
cargo build --release --target x86_64-pc-windows-gnu   # -> cn1ble.dll

# Windows arm64, cross-compiled with the llvm-mingw toolchain
# (https://github.com/mstorsjo/llvm-mingw releases; put its bin/ on PATH so
#  the aarch64-w64-mingw32-clang linker is found).
rustup target add aarch64-pc-windows-gnullvm
cargo build --release --target aarch64-pc-windows-gnullvm   # -> cn1ble.dll
```

## Validation status

| Library | Verified |
|---|---|
| macOS (universal) | Built natively; both slices present per `lipo -info` (`x86_64 arm64`). |
| Linux x64 | Built in `rust:1-bookworm` (x86_64 via emulation); ELF x86-64 shared object. |
| Linux arm64 | Built in `rust:1-bookworm` (aarch64); ELF aarch64 shared object. |
| Windows x64 | Cross-built with mingw-w64; PE32+ x86-64 DLL. Not executed. |
| Windows arm64 | Cross-built with llvm-mingw; PE32+ Aarch64 DLL. Not executed. |

A missing or broken library degrades gracefully: the JavaSE port reports the
native backend as unavailable and the simulator's virtual Bluetooth stack
keeps working.

Keep the executable bit set on the Unix libraries (`chmod +x`); the Java side
loads them from a temp file after extraction.
