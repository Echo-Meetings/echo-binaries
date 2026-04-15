# echo-binaries

Pre-built native binaries for [Echo Desktop](https://github.com/Echo-Meetings/Echo-Desktop).

This repository exists solely to host pre-compiled `whisper-cli` binaries that the
Echo Desktop app downloads at first launch. It contains no application code.

## Why a separate repo

- Echo Desktop is private. GitHub release assets on private repos require
  authentication to download — which the app cannot provide for end users.
- This repo is **public**, so asset download URLs work for every user of the
  app without any auth.
- Native binaries change very rarely (only when `whisper.cpp` is bumped),
  while the app changes constantly. Keeping the binary build pipeline in a
  separate repo means we don't re-run a 15-minute Windows build every time
  we fix a TypeScript typo.

## What gets built

The workflow in `.github/workflows/build.yml` builds two artifacts and
publishes them as release assets:

| Artifact | Platform | GPU backend | CRT |
| --- | --- | --- | --- |
| `whisper-bin-x64-windows.zip` | Windows x64 (runs on ARM64 via WoW64) | Vulkan | Static (`/MT`) |
| `whisper-bin-universal-macos.tar.gz` | macOS arm64 + x86_64 universal | Metal (embedded) | — |

The Windows build is compiled with:

- `GGML_VULKAN=ON` — Vulkan GPU acceleration
- `GGML_OPENMP=OFF` — avoids `vcomp140.dll` dependency on the VC++ redist
- `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded` + `CMP0091=NEW` — static CRT,
  avoids `MSVCP140.dll` / `VCRUNTIME140.dll` dependencies on the VC++ redist
- A verification step that runs `dumpbin /imports` on every shipped `.dll`
  and fails the build if any of them import `MSVCP140`, `VCRUNTIME140`,
  `VCRUNTIME140_1`, or `VCOMP140`

The result: whisper-cli.exe runs on a clean Windows install with zero
redistributable prerequisites.

## Triggering a build

This repo has no automatic builds. To produce a new release:

1. Go to **Actions → Build whisper binaries → Run workflow**
2. Enter a whisper.cpp version tag (e.g. `v1.8.4`)
3. Enter a release tag for this repo (e.g. `whisper-v1.8.4-vulkan`)
4. Click **Run workflow**

The workflow builds both platforms, creates a GitHub release with that tag,
and uploads both zips as release assets. The stable download URLs are then:

```
https://github.com/Echo-Meetings/echo-binaries/releases/download/<TAG>/whisper-bin-x64-windows.zip
https://github.com/Echo-Meetings/echo-binaries/releases/download/<TAG>/whisper-bin-universal-macos.tar.gz
```

Or, to always fetch the latest:

```
https://github.com/Echo-Meetings/echo-binaries/releases/latest/download/whisper-bin-x64-windows.zip
https://github.com/Echo-Meetings/echo-binaries/releases/latest/download/whisper-bin-universal-macos.tar.gz
```

## Updating Echo Desktop

After a new release is published here, update
`src/main/services/WhisperBinaryManager.ts` in Echo Desktop to point at the
new asset URLs (or leave them pointing at `releases/latest/download/...`
for automatic adoption).
