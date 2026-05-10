# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

# FinceptTerminal

C++20 desktop app — Qt 6.8.3 UI, embedded Python 3.11+ for analytics, CMake 3.27+ build system.

## Build

```bash
./setup.sh                          # first-time deps + Qt install (~3-5 min, downloads UV + Python)
cmake --preset macos-release        # configure (also: linux-release, win-release, macos-debug, linux-debug)
cmake --build build --parallel      # compile
./build/macos-release/FinceptTerminal
```

CI mode: `./setup.sh --ci`

**Hard compiler constraints** (enforced at CMake configure time with `FATAL_ERROR`):
- MSVC 19.38+ / GCC 12.3+ / Clang 15.0+ (Apple Clang = Xcode 15.2+)
- Qt **exactly** 6.8.3 — no drift

ccache/sccache is auto-detected — 5–20× speedup on incremental builds.

**No git remote configured** — verify with `git remote -v` before any push.

## Architecture

### Python Integration

Python is **not** embedded via pybind11 or ctypes. All Python execution goes through `src/python/PythonRunner`, which spawns Python scripts as async subprocesses via `QProcess` and exchanges JSON over stdout. Concurrency is capped at 3 parallel processes.

`PythonSetupManager` bootstraps the Python environment on first launch:
1. Downloads UV standalone binary (~13 MB)
2. UV installs Python 3.12.7
3. Creates **two parallel venvs** (`venv-numpy1`, `venv-numpy2`) for different analytics workloads
4. Tracks requirement staleness via SHA-256 hashes — stale envs re-install automatically

Never call Python directly from C++; always go through `PythonRunner`.

### Module Layout

| Directory | Purpose |
|-----------|---------|
| `src/screens/` | Qt UI screens |
| `src/services/` | QObject-based business logic, emit signals consumed by screens |
| `src/python/` | Subprocess IPC — PythonRunner, PythonSetupManager |
| `src/mcp/` | Bridge to external AI tools |
| `src/ai_chat/` | AI chat integration |
| `src/trading/` | Trading engine |
| `src/network/` | Network layer |
| `src/auth/` | Authentication |
| `src/storage/` | Data persistence |

Data flows via Qt signals/slots: services emit → screens react. No direct service-to-screen calls.
