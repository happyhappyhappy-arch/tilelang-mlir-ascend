# TileLang-managed NPUIR compiler

## Layout and lookup

Stage release artifacts in the source checkout before building an NPUIR wheel:

```text
3rdparty/
  bin/bishengir-compile
  bin/hivmc-a5
  lib/host.bc
  lib/meta_op.{aic,aiv,mix.aic,mix.aiv}.c220.bc
  lib/meta_op.{aic,aiv,mix.aic,mix.aiv}.c310.bc
```

The complete `bin` and `lib` directories are copied into the wheel as
`tilelang/3rdparty/bin` and `tilelang/3rdparty/lib`. Executable permissions are
preserved, and bitcode remains next to the executable's parent directory, where
the stable compiler resolves its resources. These generated source directories
are Git-ignored but included in source distributions through `MANIFEST.in`.

For A2/A3/A5, JIT compiler selection is:

1. Executable `bishengir-compile` in TileLang's `3rdparty/bin`.
2. `bishengir-compile` found through `PATH`.
3. The existing `${TILELANG_NPU_COMPILER_PATH}/npuc` fallback.
4. The existing missing-compiler error.

`tilelang.env.THIRD_PARTY_ROOT` resolves the source-checkout or installed-package
layout. A missing/non-executable bundled file falls back; a selected compiler's
compilation failure does not silently retry with a different compiler. Automatic
target detection also recognizes the bundled compiler. Lookup remains cached
per process, so restart Python after changing the compiler payload. For a bundled
compiler on A5, JIT sets `BISHENG_INSTALL_PATH` only in the compiler subprocess to
its `bin` directory, so A5 selects the matching packaged `hivmc-a5`.

The compiler comes from the unified `AscendNPU-IR` `stable` branch, pinned by
this repository's submodule commit (`8ff5928f23c37aac79123eb859522164a335adb1`
for this migration). A2/A3 and A5 share the same compiler source;
`--build-a5` still selects TileLang's A5 code generator at build time. This does
not turn an A2/A3 TileLang build into an A5 build.

The wheel includes `hivmc-a5` for the compiler's A5 backend and all nine bitcode
files. A2/A3 continue to use CANN's `hivmc`. CANN's `bisheng`, Toolkit headers and
NPU runtime remain external dependencies; source the CANN environment before use.

## Local packaging

Use a native-architecture build of the pinned NPUIR `stable` commit. The GEMM
fix is upstream; there is no TileLang-maintained NPUIR backport. The pinned LLVM
and torch-MLIR forks also include their changes already; do not run the legacy
`apply_patches.sh` or pass `--apply-patches`.
Use CMake >= 3.28 and Ninja >= 1.12 for the upstream build script. Configure
`BISHENGIR_PUBLISH=ON` to avoid passing internal options to the public CANN
`hivmc`. When building its bitcode, source CANN 9.1.0 and configure
`BISHENGIR_BUILD_TEMPLATE=ON` and `BISHENG_COMPILER_PATH` to the directory
containing CANN's `ccec` and `llvm-link`. The CI workflow shows the complete
configuration, including `BISHENGIR_ENABLE_TRITON_COMPILE=ON` required by the
unified backend. TileLang also enables the same `BSPUB_DAVINCI*` header feature
macros as the NPUIR build, for both code generators. These macros describe the
unified compiler interfaces; they do not select the runtime hardware.
The staged executable must match the wheel's architecture and
the destination system's runtime libraries.

```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh

# Run from the repository root after compiling TileLang and TVM.
# The install prefix needs both compiler executables and all nine lib/*.bc.
BISHENGIR_PATH=/path/to/stable-npuir/install bash build_wheel.sh
```

`build_wheel.sh` copies the compiler and bitcode into `3rdparty/bin` and
`3rdparty/lib`, then packages the existing TileLang/TVM libraries and MLIR Python
bindings. Without `BISHENGIR_PATH`, it uses
`3rdparty/AscendNPU-IR/build/install`.

To package manually supplied `3rdparty/bin` and `3rdparty/lib` directories,
run `USE_NPUIR=true TILELANG_SKIP_BUILD=1 python setup.py bdist_wheel` with the
prebuilt libraries and bindings available. `setup.py` rejects an NPUIR wheel
with either compiler executable missing/non-executable, or with missing/empty
required bitcode. It copies the
staged directories instead of downloading a compiler or selecting one from
the build machine's `PATH`.

## CI

NPUIR prebuild, TileLang wheel build and the wheel test job use
`quay.io/ascend/cann:9.1.0-910b-ubuntu22.04-py3.11` and the corresponding
`py3.12` image. Both x64 and arm64 builders produce Python 3.11 and 3.12 wheels.
NPUIR Python extensions are built separately for each Python version; TVM
prebuilds are shared across Python versions on the same architecture. Run steps use
`bash -l -e -o pipefail {0}`: login Bash reads `/etc/profile`, where the image
already sources Toolkit, AscendNPU-IR and NNAL. This avoids repeating `source`
commands in individual steps. GitHub Actions overrides the job container's
entrypoint, so the entrypoint alone cannot initialize these steps.
See the [runner implementation](https://github.com/actions/runner/blob/main/src/Runner.Worker/ContainerOperationProvider.cs)
and the [CANN image Dockerfile](https://github.com/Ascend/cann-container-image/blob/main/cann/9.1.0-910b-ubuntu22.04-py3.11/Dockerfile).

- The NPUIR prebuild uses the pinned upstream submodules, builds the unified
  compiler, `hivmc-a5` and bitcode using CANN's BiSheng, and caches the
  installation including `bin` and `lib`. The install step also copies embedded
  Triton source/generated headers omitted by upstream installation, allowing the
  wheel job to build TileLang against the cached install prefix alone.
- Before starting NPUIR build containers, a host job checks all four caches
  with `lookup-only`. Only missing architecture/Python combinations enter the
  build matrix. The precheck supplies the NPUIR commit even when all caches hit.
- NPUIR cache keys include architecture, Python version, the pinned commit and
  a hash of the prebuild workflow. Precheck, build and wheel jobs use matching
  keys and zstd compression. The `toolchain-v2` namespace requires an initial
  rebuild; old gzip caches are not reused. TVM restore also uses zstd to match
  the hosted builder.
- The TVM builder uses the hosted runner's existing CMake and Make, avoiding an
  unnecessary apt index refresh that can fail on unrelated third-party sources.
- Container build and packaging paths use shell `$GITHUB_WORKSPACE`, so they
  resolve inside the container rather than to the host checkout path.
- The wheel job copies the cached compiler and bitcode into `3rdparty` directly
  in its existing packaging step, then builds and uploads the wheel.
- The NPU wheel test uses matched `torch==2.10.0+cpu` / `torch-npu==2.10.0`
  and runs the existing examples and operator tests for both Python versions
  sequentially. Reports include the Python version in their artifact name.
  Release and pre-release uploads collect all four wheels after both tests pass.
