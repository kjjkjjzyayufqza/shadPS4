<!--
SPDX-FileCopyrightText: 2026 shadPS4 Emulator Project
SPDX-License-Identifier: GPL-2.0-or-later
-->

# shadPS4 Agent Guide

## Scope and sources of truth

This file applies to the repository root and all directories below it unless a more specific
`AGENTS.md` overrides it. Nested instructions in third-party submodules take precedence inside
those submodules.

Treat the repository as authoritative. In particular, consult these files before relying on a
remembered command or convention:

- `CONTRIBUTING.md` for the official C++ style.
- `README.md` and `documents/building-*.md` for supported platforms and prerequisites.
- `CMakePresets.json` and `cmake/CMake*Presets.json` for current configure presets.
- `.github/workflows/build.yml` for CI build and test expectations.
- `src/.clang-format` for formatting.

Do not edit generated build output or files under `externals/` unless the task explicitly concerns
that dependency. Most external directories are submodules and have their own upstream history.

## Project boundaries

shadPS4 is a C++23 PlayStation 4 emulator core for Windows, Linux, and macOS. The executable in this
repository is CLI-oriented; the user-facing QtLauncher is maintained in a separate repository. Keep
core changes independent of launcher behavior and local launcher layouts.

Major source areas are:

- `src/common`: shared utilities, platform support, paths, logging, and assertions.
- `src/core`: emulator lifecycle, executable loading/linking, memory, IPC, debugging, and HLE
  implementations of PS4 libraries.
- `src/core/libraries/gnmdriver`: guest GNM entry points and GPU submission plumbing.
- `src/video_core/amdgpu`: Liverpool register state, PM4 definitions, and graphics/compute command
  processing.
- `src/video_core/renderer_vulkan`, `buffer_cache`, and `texture_cache`: host Vulkan execution and
  guest/host resource synchronization.
- `src/shader_recompiler`: GCN frontend, intermediate representation and passes, and SPIR-V backend.
- `src/input`, `src/imgui`, and `src/shadnet`: input, the built-in UI, and networking support.
- `tests`: GoogleTest-based settings, GCN, and HTTP tests plus their stubs.

Preserve these architectural rules from `CONTRIBUTING.md`:

- Do not introduce new external dependencies into Core.
- Do not add platform-specific code to Core; isolate it behind the existing platform abstractions.
- Prefer a general emulation or hardware-semantics fix over a title-specific workaround.
- Do not remove required game content, disable subsystems, or change user configuration to hide a
  regression.

## Working method

1. Inspect the current branch, worktree, submodule state, and relevant configuration before editing.
   Existing changes belong to the user; preserve unrelated work.
2. Establish a reproducible baseline. Record the executable, build type, configuration, GPU, game
   state, elapsed time, and exact liveness or failure signal.
3. Trace the narrowest authoritative path from the guest API through the emulated subsystem. Check
   data ownership, queue state, synchronization, and protocol invariants before changing behavior.
4. Search upstream issues, pull requests, commits, and relevant forks for existing investigations or
   fixes before implementing a new solution. Reuse sound prior work when its hardware semantics,
   license, and regression risk fit this codebase; record why any similar approach is insufficient.
5. Test one causal hypothesis at a time. Instrument only the state needed to prove or disprove it,
   and revert falsified experiments completely.
6. Implement the smallest general fix that preserves the real protocol semantics. Add comments for
   non-obvious invariants and reasons, not line-by-line narration.
7. Verify formatting, build, tests, runtime behavior, and the final diff. Remove temporary logs,
   debugger scripts, probes, and title-specific instrumentation before handing off.

Do not infer a root cause from the severity or color of a log message. Repeated stub or unsupported
API messages may be a useful liveness marker while still being only a downstream symptom.

## Build and test

Initialize dependencies after cloning:

```sh
git submodule update --init --recursive
```

On Windows, use the supported x64 Visual Studio/Build Tools environment with `clang-cl`, CMake, and
Ninja. ARM64 and MSYS2 builds are not supported. The standard Debug workflow is:

```sh
cmake --preset x64-Clang-Debug
cmake --build Build/x64-Clang-Debug --target shadps4 --parallel
```

Use `x64-Clang-Release` or `x64-Clang-RelWithDebInfo` only when that build type is relevant. Follow
the platform document rather than copying Windows-specific commands to Linux or macOS.

Configure tests explicitly because they are disabled by default:

```sh
cmake --preset x64-Clang-Debug -DENABLE_TESTS=ON
cmake --build Build/x64-Clang-Debug --parallel
ctest --test-dir Build/x64-Clang-Debug --output-on-failure
```

CI excludes `GcnTest` from its ordinary CTest pass; run it deliberately when shader translation or
GCN behavior changes. Add a focused unit test when a stable seam exists. For scheduler, GPU, video,
or timing fixes that cannot be represented faithfully by a unit test, document and perform a
targeted runtime regression test as well.

Format every changed C++ source/header with Clang Format 19 using `src/.clang-format`. A useful
pre-handoff check is:

```sh
clang-format-19 --dry-run --Werror <changed-cpp-files>
git diff --check
```

The CI formatting script also rejects trailing whitespace. Keep SPDX headers on new source and
documentation files and follow the repository's REUSE conventions.

## C++ conventions

- Use four spaces, never tabs, and target a 100-column line width.
- Use `PascalCase` for functions and classes, `lower_case_underscored` for variables and files, and
  `PascalCase` for namespaces.
- Initialize members and pointers. Prefer uniform initialization where it remains readable.
- Use C++ casts. Avoid `dynamic_cast`, and use `const_cast` only for an external
  const-incorrect API.
- Sort includes as demonstrated in `CONTRIBUTING.md`: standard library, external libraries, then
  project headers, with meaningful module groups.
- Prefer explicit ownership and bounded spans. Audit every pointer or reference that survives a
  container mutation, coroutine suspension, asynchronous callback, or queue handoff.
- Use assertions for internal invariants, not for recoverable guest input. Include enough state in
  an assertion or error to diagnose the violated invariant.
- Comments are required where hardware semantics, synchronization, lifetime, or an intentional
  workaround would otherwise be easy to simplify incorrectly. Explain why the code exists and what
  must remain true.
- Avoid speculative abstractions, broad refactors, and unrelated cleanup in a bug-fix change.

## Emulator debugging discipline

### Reproduction and liveness

- A process being alive or a window reporting as responsive does not prove guest progress.
- Define a hang using at least one guest-progress signal: advancing frames, changing command/fence
  values, input response, or continued expected log/state transitions.
- Observe a successful candidate well past the old failure interval and through the same gameplay
  path. Starting the title or reaching the first frame is not sufficient.
- Keep media, input, patches, and configuration equivalent to the failing case unless one of them is
  the variable under test.

Use the CLI's `-g/--game`, `--override-root`, `--wait-for-debugger`, and `--wait-for-pid` options to
make test runs explicit and isolated. Prefer a temporary override root for experiments that should
not mutate the user's normal data directory.

### Evidence collection

- Capture the main thread and all relevant worker threads from the live hung process. Correlate a
  guest wait loop with the producer that should advance it.
- Inspect actual queue indices, read/write offsets, packet lengths, fence destinations and values,
  task/coroutine state, and ownership. A plausible stack alone is not proof.
- A manual memory or fence write is a diagnostic intervention only. If it releases a wait, it proves
  the wait is causal; it does not by itself explain why the producer failed.
- Hardware-vendor differences are valuable timing or semantics evidence, not permission to add a
  vendor-specific workaround without a demonstrated host-driver requirement.
- Disable overlays or implicit Vulkan layers in the test process when isolating interference, but do
  not turn that isolation setting into the product fix unless evidence identifies the layer.
- Keep diagnostic logging rate-limited and temporary. High-volume logging can change scheduling and
  performance enough to mask or create timing failures.

On Windows, debugger command files are more reliable than long inline command strings when paths,
symbols, or multiple commands are involved. Detach cleanly after read-only inspection so the same
process can continue or be terminated intentionally.

## GPU, PM4, and queue invariants

PM4 is the AMD command packet protocol consumed by the emulated Liverpool command processor. Guest
GNM calls expose portions of graphics and asynchronous-compute ring buffers; a submission boundary
is not necessarily a packet boundary.

When changing `sceGnmDingDong`, `Liverpool::ProcessCompute`, or related queue code, preserve all of
these invariants:

- Ring wrap and DingDong updates can split one PM4 packet across two or more submitted spans.
- Type-2 padding consumes one DWORD. For Type-3, `NumWords()` is the body size and total consumption
  includes the header DWORD.
- Do not assume a packet fits a fixed scratch buffer or completes in the next submission. Real
  command streams can leave more than 1024 DWORDs buffered before the final fragment arrives.
- Accumulate incomplete packets until the complete declared packet is available, then parse them.
  Never reinterpret or execute a partial packet body.
- Advance the guest read pointer by the number of DWORDs consumed from the current ring fragment,
  not by the full logical packet size when earlier fragments were already consumed.
- Reacquire pointers after a `std::vector` insertion or any operation that may reallocate storage.
- Queue-owned pending data must remain valid across coroutine yields, but must not leak into another
  virtual queue or an unrelated indirect buffer.
- Keep queue scheduling and fence signaling ordered according to guest semantics. A host callback or
  Vulkan completion must not outlive captured guest memory, queue objects, or renderer resources.

For fence hangs, trace both sides: the guest consumer waiting on a value and the PM4 producer that
should execute `ReleaseMem`, `WriteData`, or another signaling operation. Validate that the command
was submitted, parsed, executed, and made visible before modifying cache or interrupt behavior.

## HLE and stub behavior

- A frequently logged `(STUBBED)` call is not automatically the missing implementation responsible
  for a hang. Confirm that the guest depends on its output or side effect.
- Preserve ABI signatures, return codes, structure layout, and registration metadata when adding an
  HLE implementation.
- Do not convert an error log to a success path merely to silence output. Change log severity only
  when the implementation semantics justify it.
- Unsupported time, SharePlay, video, and input calls can reveal which guest threads remain alive;
  correlate them with the stopped subsystem before treating them as causes.

## Verification matrix for runtime fixes

At minimum, record evidence for the layers touched by the change:

- Clean compile of the affected target in Debug.
- Clang-format and whitespace checks for all changed C++ files.
- Relevant unit/CTest targets when available.
- Reproduction with the same content and configuration that previously failed.
- Runtime beyond the former failure window, including frame progression and input response.
- No new crash, assertion, unbounded log spam, or material performance regression.
- A final diff containing only the general fix, necessary comments/tests, and intended
  documentation.

If the problem varies by GPU vendor, test another vendor when hardware is available. When it is not,
state the limitation rather than generalizing from one device.

## Git and handoff

- Use short-lived branches such as `fix/<concise-description>` for bug fixes.
- Keep commits focused and write an imperative message that explains the subsystem and outcome.
- Do not rewrite shared history, force-push, push a branch, or open a pull request unless explicitly
  requested.
- Before committing, inspect `git status`, `git diff`, and the staged diff. Do not stage logs,
  dumps, game files, user data, local paths, IDE state, or build artifacts.
- In the handoff, report the branch, commit(s), build/test commands, runtime evidence, and any
  remaining platform coverage limits.

## Definition of done

A change is complete only when the causal defect is fixed without hiding required behavior, the
worktree contains no accidental diagnostics or artifacts, official formatting/build checks pass,
and verification covers the original failure mode. A surviving process, reduced logging, or one
successful startup is insufficient evidence on its own.
