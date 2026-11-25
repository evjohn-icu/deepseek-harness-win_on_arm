# DeepSeek Harness · win-arm64 (unofficial community build)

> **Unofficial community port.** This fork adds a native **Windows ARM64** desktop build target to
> [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness). It is not affiliated
> with or endorsed by DeepSeek. The upstream project is MIT-licensed; this port keeps its LICENSE.
> Official downloads remain at the upstream repository (macOS arm64 / macOS x64 / Windows x64).

## Download

Grab the installer from the [latest prerelease](https://github.com/evjohn-icu/deepseek-harness-win_on_arm/releases/latest):

`deepseek-harness-<version>-win-arm64-unsigned.exe` (~285 MB, NSIS, per-user install).

It is **unsigned**: Windows SmartScreen will warn ("More info" → "Run anyway"), and the app has no
automatic-update feed. Everything the official Windows build does is expected to work; see
[Known limitations](#known-limitations).

Prefer to build it yourself? See [Building it yourself](#building-it-yourself).

## What this branch changes

The upstream packaging pipeline only ships `mac-arm64`, `mac-x64` and `win-x64`. This branch
(16 files, one commit) adds the fourth target `win-arm64`:

- **Target plumbing** — `desktop-build-paths`, `package-target`, `desktop-upload-plan`,
  `desktop-build-version-discovery` accept and map `win-arm64`; the host check requires any
  Windows machine (native ARM64 build host, not cross-compilation).
- **Update policy** — `mandatory-update-policy` no longer rejects non-x64 Windows hosts, and
  `electron-builder-config` omits mandatory-update metadata from arm64 artifacts because the
  release service does not publish `x-client-arch: arm64` records.
- **Primary runtime** — Node zip extraction and platform mapping handle every `win-*` target;
  `lock.json` pins Node 24.21.0, CPython 3.12.14 (`install_only_stripped`,
  python-build-standalone 20260901) and numpy/pandas/pillow/lxml `cp312 win_arm64` wheels
  with SHA-256. Note: the win-arm64 lock entry hashes the stripped CPython variant, so the
  cache-key/URL mismatch in the original kit is resolved by design here.
- **NSIS installer** — `installer.nsh` resolves `installer-ui` assets per target through the
  `APP_ARM64` define, and `path.nsh` picks the matching unpacked-size define for the disk-space
  preflight.
- **Publisher types** — `desktop-publisher` admits `win-arm64` in its target set and
  declarations; CI retention explicitly refuses arm64 until a publish route exists.
- **Scripts** — `package:win:arm64[:unsigned|:dir]` on desktop and root `package.json`.

## Building it yourself

Prerequisites (native ARM64 Windows 11 host):

| Tool | Version |
| --- | --- |
| Node.js | 24.x arm64 |
| pnpm | 11.28.5 (`corepack enable` or `npm i -g pnpm@11.28.5`) |
| Git | any recent |
| Python | 3.12 arm64 on PATH or `PYTHON` set |
| VS Build Tools | "MSVC v143 x86/x64 C++" + Windows SDK (compiles the x86 installer helper via `vcvars32`) |
| PowerShell | 7.x (`pwsh`) |

```powershell
pnpm install
# first run creates apps/desktop/.env.windows from the template values below
pnpm run package:desktop:win:arm64:unsigned            # NSIS installer
pnpm run package:desktop:win:arm64:dir                 # unpacked directory
pnpm run package:desktop:win:arm64:unsigned -- --check # toolchain/config check only
```

Artifacts land in `apps/desktop/.desktop-build/targets/win-arm64/unsigned-artifacts/`.
Verify the PE machine type if in doubt: the unpacked `DeepSeek Harness.exe` must report `0xAA64`.

`pnpm install` needs network access to registry.npmjs.org (or `DSH_DESKTOP_NPM_REGISTRY=https://registry.npmmirror.com`),
and the runtime prepare step downloads from nodejs.org, GitHub releases and pythonhosted.org
(honors standard `ELECTRON_MIRROR` / `ELECTRON_BUILDER_BINARIES_MIRROR` variables).

### CI

`.github/workflows/win-arm64-package.yml` builds on GitHub's hosted `windows-11-arm` runner:

- `workflow_dispatch` — artifacts attached to the run.
- pushing a tag `win-arm64-build-v*` — publishes a prerelease with the installer.

## Known limitations

- **Unsigned** — SmartScreen prompts on first run; no automatic update feed.
- **No forced-update channel** — arm64 is invisible to the official update service; the app
  never blocks on a missing policy record (this is deliberate).
- **Voice input** — `sherpa-onnx` ships no win-arm64 package, so speech features may be absent.
  Everything else works.
- **Third-party plugins** carrying x64-only native modules will fail to load.

## Verified

Built and smoke-tested on a Snapdragon X Elite (X1E78100) host with Node 24.21.0 arm64:
packaged-runtime smoke passes with native `koffi`, `sharp`, `node-pty`, `pnpm`, `ripgrep`, and
the Office conversion skills (DOCX/XLSX/PPTX to PDF, rendering, recalculation) all pass.
