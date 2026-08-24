# Building MerMark Editor

This guide describes how to install dependencies, run local checks, build the web assets, and package the Tauri desktop application.

## Prerequisites

Install these tools before building:

- [Node.js](https://nodejs.org/) 18 or newer.
- [pnpm](https://pnpm.io/) 11.x. The repository declares `pnpm@11.3.0` in [package.json](../package.json).
- [Rust](https://rustup.rs/) stable toolchain.
- Windows desktop build tools when building on Windows:
  - Microsoft C++ Build Tools or Visual Studio with the Desktop development with C++ workload.
  - WebView2 Runtime. Most Windows 10 and Windows 11 systems already include it.

For a normal local build, no signing keys are required. Signing keys are only needed when creating updater artifacts for release distribution.

## Fresh Checkout Setup

From the repository root:

```powershell
pnpm install
```

This installs the Vue, Vite, Tauri CLI, TypeScript, and test dependencies from [pnpm-lock.yaml](../pnpm-lock.yaml).

## Frontend Web Build

Run the frontend production build first:

```powershell
pnpm build
```

This command runs:

```powershell
vue-tsc --noEmit && vite build
```

Expected output:

- TypeScript type checking completes without errors.
- Vite writes production assets to `dist/`.

You may see Vite warnings about large chunks or modules that are both statically and dynamically imported. These warnings do not fail the build.

## Desktop App Build

To build the Tauri desktop application for local use:

```powershell
pnpm tauri build --config '{"bundle":{"createUpdaterArtifacts":false}}'
```

This creates an unsigned local build and skips updater artifact signing. Use this command when you do not have the private updater signing key.

Expected Windows outputs:

- Raw executable: `src-tauri/target/release/mdreader.exe`
- NSIS installer: `src-tauri/target/release/bundle/nsis/MerMark Editor_0.7.0_x64-setup.exe`
- MSI installer, English: `src-tauri/target/release/bundle/msi/MerMark Editor_0.7.0_x64_en-US.msi`
- MSI installer, Polish: `src-tauri/target/release/bundle/msi/MerMark Editor_0.7.0_x64_pl-PL.msi`

The exact version number in file names comes from [src-tauri/tauri.conf.json](../src-tauri/tauri.conf.json) and [package.json](../package.json).

## Release Build With Updater Artifacts

The default desktop build is:

```powershell
pnpm tauri build
```

The project has updater artifacts enabled in [src-tauri/tauri.conf.json](../src-tauri/tauri.conf.json):

```json
"bundle": {
  "createUpdaterArtifacts": true
},
"plugins": {
  "updater": {
    "pubkey": "..."
  }
}
```

Because a public updater key is configured, Tauri expects the matching private key when creating updater artifacts. Set it before running the release build:

```powershell
$env:TAURI_SIGNING_PRIVATE_KEY = "<private-key>"
$env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD = "<private-key-password>"
pnpm tauri build
```

Do not commit private keys, passwords, or generated key files. The repository ignores `*.key` files, except public `*.key.pub` files.

If you run `pnpm tauri build` without the private key, the application and installers may still be produced, but the command exits with an error similar to:

```text
A public key has been found, but no private key. Make sure to set `TAURI_SIGNING_PRIVATE_KEY` environment variable.
```

Use the unsigned local build command if you only need a local installer.

## Development Server

For frontend-only development:

```powershell
pnpm dev
```

For the full Tauri development app:

```powershell
pnpm tauri dev
```

The Tauri dev configuration starts Vite through the `beforeDevCommand` in [src-tauri/tauri.conf.json](../src-tauri/tauri.conf.json).

## Tests

Run the unit and component test suite once:

```powershell
pnpm test:run
```

Run Vitest in watch mode:

```powershell
pnpm test
```

Run Playwright end-to-end tests:

```powershell
pnpm test:e2e
```

## Clean Build Outputs

Generated build outputs are ignored by Git:

- `dist/`
- `src-tauri/target/`

To force a clean rebuild, delete those directories and rerun the build commands:

```powershell
Remove-Item -Recurse -Force dist, src-tauri/target
pnpm build
pnpm tauri build --config '{"bundle":{"createUpdaterArtifacts":false}}'
```

## Troubleshooting

### `pnpm` Is Not Recognized

Install pnpm or enable Corepack:

```powershell
corepack enable
corepack prepare pnpm@11.3.0 --activate
```

Then rerun:

```powershell
pnpm install
```

### Rust Or Linker Errors On Windows

Make sure Rust and the Microsoft C++ build tools are installed, then restart the terminal so PATH changes are picked up.

Check Rust with:

```powershell
rustc --version
cargo --version
```

### Updater Signing Error

For local builds, use:

```powershell
pnpm tauri build --config '{"bundle":{"createUpdaterArtifacts":false}}'
```

For release builds, set `TAURI_SIGNING_PRIVATE_KEY` and, if required, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`.

### Large Chunk Warnings

Vite may report chunks larger than 500 kB after minification. This is a build warning, not a build failure. It can be addressed later with code splitting or Rollup `manualChunks` configuration if bundle size becomes a release priority.