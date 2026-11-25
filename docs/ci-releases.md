# CI-built SDK archives

The **Build OpenGL 4.6 SDK** GitHub Actions workflow builds a fresh developer SDK
and a showcase app ZIP on Ubuntu 24.04. It runs on pushes and pull requests
targeting `main`, version tags and manual requests. The default `dynamic`
profile starts at 1080p60 and supports up to 2160p120 at runtime. Manual runs
also offer fixed 1080p60, 1440p120 and 2160p120 profiles. The fresh binaries
are verified on the build host.
Historical full CTS, sampled CTS, performance and stability results apply only to
their recorded binary identities, not automatically to these downloads.

## CPU and software-OpenGL CI checks

**Host checks** retains its independent `make test` job and adds a staging job on
Ubuntu 24.04, bounded to 20 minutes overall and 10 minutes for `make test-staging`.
**Build OpenGL 4.6 SDK** also requires `make test-staging` to pass, with a 10-minute
step limit, after fetching sources and before building the SDK. Both workflows
retain their existing `make test` checks and read-only test/build permissions.

The staging lane installs `build-essential`, `clang-18`, the GCC ASan/UBSan
runtimes (`libasan8`, `libubsan1`), EGL/GL development headers (`libegl-dev`,
`libgl-dev`), and system Mesa (`libegl-mesa0`, `libgl1-mesa-dri`, `libglx-mesa0`).
It uses `python3 tools/fetch-sources.py` for the revisions and Mesa archive checksum
in `dependencies.json`; the release workflow keeps its existing `--sokol-samples`
option. No CTS fetch or PS5 SDK build is needed for this lane.

`PS5_OPENGL_PREFIX` points to `third_party/mesa-26.2.0` for the staging script's
headers. The executables link the system `libEGL.so.1` and `libGL.so.1`, run
surfaceless with `LIBGL_ALWAYS_SOFTWARE=1`, `GALLIUM_DRIVER=llvmpipe` and two
renderer threads, and clear library/driver/vendor overrides before execution.
No PS5 driver is injected. UBSan findings stop execution. The existing test
scripts enforce CPU assertions, sanitizer checks, pixel/API oracles and their
deliberate-fault rejection checks; failures and timeouts fail the CI lane.

On a Linux/WSL host with those packages already available, reproduce the staging
step without building an SDK:

```sh
python3 tools/fetch-sources.py
unset LD_LIBRARY_PATH LD_PRELOAD LIBGL_DRIVERS_PATH
unset MESA_LOADER_DRIVER_OVERRIDE __EGL_VENDOR_LIBRARY_FILENAMES __EGL_VENDOR_LIBRARY_DIRS
PS5_OPENGL_PREFIX="$PWD/third_party/mesa-26.2.0" \
  EGL_PLATFORM=surfaceless LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  LP_NUM_THREADS=2 MESA_GL_VERSION_OVERRIDE=3.3 MESA_GLSL_VERSION_OVERRIDE=330 \
  MESA_SHADER_CACHE_DISABLE=true UBSAN_OPTIONS=halt_on_error=1 \
  timeout 10m make test-staging
```

Passing this lane covers the selected CPU helpers and software-GL reference
oracles, not native GPU execution, cache coherence, independent MSAA sample
isolation or console qualification. Native-only tests are not run or counted as
passes. The separate `make test-depth-targets` is not part of this lane: system
Mesa can report layer zero when the 1D-array attachment selected layer two. Its
strict query oracle still exits 1 for that known discrepancy; it is not waived,
relabelled as PASS or hidden with `continue-on-error`. See [testing](testing.md)
for the individual checks and native coverage limits.

## Download and use

Use the workflow's **Run workflow** button to build any selected project branch.
Download the `ps5-opengl-4.6-sdk` Actions artifact. GitHub wraps the SDK and
showcase packages and their checksum files in a single ZIP for download:

- `ps5-opengl-sdk-<version>.tar.gz`
- `ps5-opengl-sdk-<version>.tar.gz.sha256`
- `ps5-opengl-showcase-<version>-PPSA99005.zip`
- `ps5-opengl-showcase-<version>-PPSA99005.zip.sha256`

```sh
sha256sum --check ps5-opengl-sdk-*.tar.gz.sha256
tar -xzf ps5-opengl-sdk-<version>.tar.gz
cd ps5-opengl-sdk-<version>
sha256sum --check SHA256SUMS
export PS5_OPENGL_PREFIX="$PWD/sdk"
```

The archive contains `sdk/` (EGL/GL/KHR headers, 17 regular static archives,
an umbrella linker script, two generated import stubs, Make/pkg-config/CMake
integration), `examples/`, `docs/`, licenses, complete project/dependency source
archives, `consumer-validation.json`, `runtime-config.txt` and `provenance.json`.
The public payload SDK remains a separate prerequisite; it and firmware modules
are not redistributed. No ready-to-launch application or console controller is
included. Native application assembly still uses the pinned boilerplate; see
`docs/consumer-build.md` and `docs/building.md` inside the archive.

The sources include upstream archives plus this project's build files and patches.
Debug information is retained and can contain build-runner paths. Source and
dependency identities are recorded; byte-identical rebuilds across runner/tool
updates are not promised. Compiler versions and SDK hashes are in the consumer
report. Selecting an HFR profile does not confer the frozen G47 binaries'
console qualification.

## Maintainer release process

1. Push the intended source revision to `main` and wait for **Build OpenGL 4.6 SDK**.
2. Inspect host/compiler checks, all 657 Core exports, three relocated consumer
   links, SDK and showcase checksums and the recorded build configuration.
3. Set the showcase's `contentVersion` for the release and push a tag `vX.Y.Z`
   on the checked commit. The same workflow builds
   from that tag and publishes the release, marked Latest. If a release for the
   tag already exists (notes written ahead, or a draft), the workflow adds the
   files it does not have yet and leaves its title, notes and files as they are.
4. If console validation is performed, retain receipts for these exact bytes and
   identify that scope explicitly. Do not attach an older SDK's acceptance to a
   rebuild.

Every successful run keeps the SDK archive and the showcase app ZIP as Actions
artifacts for seven days. The showcase app ZIP
stores every entry with permissions 0777 (`tools/zip-open-modes.py`), because the
console only starts an app whose files are open to all; the build fails otherwise.
The SDK archive keeps its ordinary permissions.

The SDK archive and the showcase app ZIP built by the workflow can be checked with
`gh attestation verify <file> -R BlackBearRedux/ps5-opengl` (GitHub CLI).

Manual runs only create Actions artifacts (seven-day retention); they do not
create tags or releases. A release's files are the ones the workflow run for its
tag built and attested: the SDK archive, the showcase app ZIP and their checksum
files; none is attached by hand. Existing releases/assets are never overwritten:
a file a release already has is neither replaced nor removed, and the run prints
a warning that it did not come from this run and may have no attestation, then
ends green. Repository visibility is not
changed by this workflow. Access follows the repository's permissions.

The build uses pinned public source revisions, the hash-verified payload SDK
v0.42, LLVM 21 from its signed upstream Ubuntu repository, and versioned Python
build tools. GitHub Actions are commit-pinned; build jobs have read-only repository
permission and no console access. Only the separate release job can write
release assets. The older sample-validated bundle remains a separate frozen
artifact, documented in `docs/sdk-bundle.md`.

Frozen releases such as [SDK 0.2.0](release-g62.md) use `sdk-*` tags and
preverified assets rather than this fresh-build workflow. Their archives retain
their original runtime/build provenance and exact-binary acceptance.
A source push or CI pass does not replace those downloads or qualify a new binary.
