# WinSAR

## Overview

WinSAR is a free archive manager for Windows 11, written in Rust. It works as a file explorer in which an archive opens like a folder, and it integrates with Windows File Explorer through the Windows 11 context menu and the preview pane. It also ships a command-line tool.

## Problem

Archive tools on Windows tend either to keep an interface that predates Windows 11 or to integrate only through the legacy context menu. Extraction is also a classic attack surface: an entry named `../../something` or with an absolute path can write outside the destination folder. WinSAR's goals are to open archives as ordinary folders, to integrate with the current Windows shell, and to make extraction safe by construction.

## Approach / Architecture

The repository is a Cargo workspace with four crates and a Tauri application:

| Component | Role |
|---|---|
| `crates/winsar-core` | Archive engine: one backend per format (ZIP, 7z, TAR family, RAR), safe extraction, creation, modification, conversion, split volumes, integrity test |
| `crates/winsar-cli` | `winsar` binary: `list`, `extract`, `create`, `convert`, `test` |
| `crates/winsar-shell` | DLL loaded by `explorer.exe` that implements `IExplorerCommand` for the Windows 11 context menu |
| `crates/winsar-preview` | DLL that implements `IPreviewHandler` to show an archive's content in the Explorer preview pane |
| `app/src-tauri`, `app/src` | Tauri 2 desktop application; interface in TypeScript without a framework |
| `packaging` | Sparse MSIX manifest and scripts that register the shell integration |

**Formats.** ZIP, 7z, TAR, TAR.GZ, TAR.BZ2, TAR.XZ and TAR.ZST can be read and written. RAR is read-only, through the UnRAR library under its own license.

**Safe extraction.** Every entry name goes through a single function before anything is written: any `..` component rejects the entry, absolute paths and drive letters are stripped, characters forbidden by Windows are replaced, and reserved device names (`CON`, `NUL`, `COM1`…) are prefixed. When an archive is created, symbolic links are skipped so that nothing outside the selection is archived.

**Encryption.** ZIP archives can be encrypted with AES-256. 7z archives use AES-256 and also encrypt the header, so file names are not readable without the password.

**Other operations.**

- Adding and removing files; in ZIP, kept entries are copied without recompression.
- Converting between formats: the content is extracted to a hidden working folder next to the destination and re-archived, and the original is never modified.
- Split volumes `.001`, `.002`… in the 7-Zip format, an integrity test, CRC32 and SHA-256 checksums.
- Opening nested archives in place.

**Interface.** The interface provides a home view with quick access and drives, an address bar that goes from the disk into an archive, history, search, and drag and drop in both directions. The list is virtualised so that it can show tens of thousands of entries.

## Tech stack

Rust (zip, sevenz-rust2, tar, flate2, xz2, zstd, unrar, windows-rs for COM) · Tauri 2 · TypeScript · NSIS installer · sparse MSIX packaging · GitHub Actions.

## How to run

Installers are on the [releases page](https://github.com/moha528/winsar/releases), each with its SHA-256 sums and an entry in the [changelog](CHANGELOG.md).

Releases are not code-signed yet. Windows SmartScreen may therefore warn at installation: choose "More info", then "Run anyway". For the same reason the installer cannot register the Windows 11 context menu or the preview handler; it installs the classic context menu instead, under "Show more options" (Shift+F10). See [packaging/README.md](packaging/README.md).

To build from source you need Windows 10 21H2 or later (Windows 11 recommended), stable Rust with the `x86_64-pc-windows-msvc` toolchain, Visual Studio Build Tools 2022 with the "Desktop development with C++" workload, Node.js 22 or later, and WebView2.

```bash
cd app && npm install
cd app && npm run tauri dev
cd app && npm run bundle
```

Tests of the engine, the context menu and the preview handler, and the command-line tool:

```bash
cargo test --workspace
cargo run -p winsar-cli -- extract archive.7z
```

## Status

- Version 0.1.0, released on 22 October 2026.
- 33 tests: unit tests (including path traversal and Windows name sanitisation) and integration tests that round-trip every format with and without AES, check modification, split volumes, corruption detection, wrong-password reporting, cancellation and read-only RAR, and load the shell and preview COM components.
- Continuous integration on Windows runs Clippy with warnings as errors and the full test suite.
- Releases are unsigned until the code-signing application described below is accepted.

Contributions are welcome: see [CONTRIBUTING.md](CONTRIBUTING.md). The code and its comments are written in French.

## License

[MIT](LICENSE). RAR extraction relies on Alexander Roshal's UnRAR code, under its own license: see [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md).

## Code signing policy

Free code signing for Windows releases will be provided by [SignPath.io](https://about.signpath.io/), with a certificate by the [SignPath Foundation](https://signpath.org/) (application in progress). Until then, releases are unsigned.

- Every signed file is built from this repository by the [release workflow](.github/workflows/release.yml) on GitHub Actions, never on a personal machine.
- Committers and reviewers: [moha528](https://github.com/moha528). Approvers: [moha528](https://github.com/moha528).
- Signing requests are approved manually, for tagged releases only.

## Privacy

WinSAR does not collect, store or transmit personal data. It sends no telemetry and has no account.

The only network request it makes is the update check: at most once a day, WinSAR reads the latest release from the public GitHub API (`api.github.com/repos/moha528/winsar/releases/latest`). Nothing about you or your files is sent; GitHub sees an ordinary request from your IP address, as for any visit. You can turn it off in Settings › Updates.
