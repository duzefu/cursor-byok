# CLAUDE.md

## Building the Windows desktop app on Linux (MSVC)

Cross-compile `x86_64-pc-windows-msvc` with `cargo-xwin`, the same toolchain family as the
`windows-latest` release job. The output links the VC runtime and WebView2Loader statically, so it
runs without extra DLLs. Prefer this over `x86_64-pc-windows-gnu`, which needs
`WebView2Loader.dll` shipped next to the exe.

One-time setup:

```bash
sudo apt install -y nsis lld clang-19 libayatana-appindicator3-dev
rustup target add x86_64-pc-windows-msvc
cargo install --locked cargo-xwin
# cargo-xwin calls an unversioned `clang-cl`; the MSVC STL it downloads requires Clang >= 19.
mkdir -p ~/.local/share/cursor-byok-xwin/bin
ln -sf /usr/lib/llvm-19/bin/clang ~/.local/share/cursor-byok-xwin/bin/clang-cl
```

Build (from `apps/desktop`):

```bash
PATH=~/.local/share/cursor-byok-xwin/bin:$PATH npx tauri build \
  --runner cargo-xwin --target x86_64-pc-windows-msvc --bundles nsis \
  --config '{"bundle":{"createUpdaterArtifacts":false}}'
```

Outputs:

- `target/x86_64-pc-windows-msvc/release/bundle/nsis/Cursor BYOK_<version>_x64-setup.exe`: the installer
- `target/x86_64-pc-windows-msvc/release/cursor-byok-desktop.exe`: the portable exe

Notes:

- The first build downloads the MSVC CRT and Windows SDK (~1 GB) into `~/.cache/cargo-xwin`, which
  accepts Microsoft's license.
- `createUpdaterArtifacts` is disabled because local builds lack the release signing key. These
  builds are unsigned and in-app updates will not accept them.
- `libayatana-appindicator3-dev` is required even for Windows targets: when it runs on Linux,
  tauri-cli's bundler probes the Linux tray library through `pkg-config` and panics with
  `Can't detect any appindicator library` if it is missing.
