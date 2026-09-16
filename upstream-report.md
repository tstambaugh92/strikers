# Linux cup crash and build portability findings

## 1. Cup completion crash on LP64 platforms

After the third match of an early cup, the game crashed while returning to the cup standings screen. The original log and GDB backtrace are in [`crash-log.txt`](crash-log.txt).

### Cause

`PlayerStats` declared these counters as `unsigned long`:

```cpp
unsigned long mBallPossessionTime;
unsigned long mNumButtonPresses;
```

They are intended to be 32-bit. On Linux and macOS, `unsigned long` is 64-bit, making `PlayerStats` 64 bytes instead of `0x34` and `TeamStats` 80 bytes instead of `0x40`.

The final-round path in `GameInfoManager::OnPostCupGameState()` copies a `TeamStats` through a 64-byte scratch buffer. The release binary copied 80 bytes into it, overwriting the saved `rbx` register. On return, `TransitionTask::InitializeFEState()` used the corrupted `rbx` as its `this` pointer and faulted at its final `m_TransitionState` assignment.

This should affect LP64 Linux builds, including Ubuntu, and potentially macOS. Windows uses a 32-bit `unsigned long` and is not affected by this layout error.

### Fix

Commit `440c885` changes both fields to `u32` and adds size assertions for `PlayerStats` and `TeamStats`.

The corrected build completed the previously failing cup sequence.

### Save compatibility

Existing Linux/macOS saves written by a build with the incorrect 80-byte `TeamStats` layout may be rejected as the wrong size by a corrected build. A migration path may be needed if preserving those saves is important.

## 2. Linux build portability issues

Commit `13f88f2` contains separate build-system fixes found while rebuilding on Arch Linux with Clang 22.

### Clang 22 and `wcslen`

The project uses `-fshort-wchar` for 16-bit game strings. Clang 22 optimized the manual `nlStrLen<unsigned short>` loop into glibc `wcslen()`, whose Linux ABI uses 32-bit `wchar_t`. This returned an incorrect length, under-allocated a stack buffer, and caused a pre-title-screen crash.

The Linux build now uses `-fno-builtin-wcslen`. The Ubuntu 22.04/Clang 18 release environment may not currently trigger this optimization, but newer toolchains can.

### Stub generation

`genstubs.py` recursively scanned fetched dependency source trees and attempted to inspect Dawn's intentionally corrupt LLVM test archives. It now excludes FetchContent `*-src` directories while retaining generated dependency archives.

### System `fmt`

On Arch, CMake selected a shared system `fmt`. Because `genstubs.py` discovers dependency definitions from static archives, it mistook four `fmt` symbols for missing game functions. `configure.sh` now forces Aurora's bundled static `fmt`, consistent with the existing hermetic dependency approach.

## Validation

- Full Release build and link succeeded.
- `python3 tools/test_genstubs.py`: 57 cases passed.
- A startup smoke test passed the previous Clang 22 crash point.
- The original three-match cup crash was reproduced before the fix and confirmed resolved afterward.
