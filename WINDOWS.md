# Windows fork — office-3d 1.4.2-win.1

Local fork of `oslook/hermes-desktop-plugin-office-3d` @ v1.4.1 (MIT), patched by
Hermes/TechZilla to run on Windows. Upstream declares `platforms: [linux, macos]`
and README §Requirements says "Windows support has not been validated".

## What was actually blocking Windows

Only one code path. Both `__init__.py` and `dashboard/plugin_api.py` carried:

```python
if os.name == "nt":
    return False          # refuse: os.kill(pid, 0) would KILL the target
```

`os.kill(pid, 0)` on Windows is routed through `TerminateProcess`, so a bare
liveness probe would terminate the gateway it is asking about. Upstream chose to
refuse the check on Windows; everything else (profile discovery, `state.db`
read-only aggregates, gateway resolution, the plugin SDK surface) is portable.

## The patch

1. **`_windows_pid_alive()`** added in both files: `OpenProcess(
   PROCESS_QUERY_LIMITED_INFORMATION)` + `GetExitCodeProcess` via `ctypes` —
   the same technique Hermes' own `pid_exists_stdlib` uses, kept local so the
   plugin imports no private core module. Never signals the target.
   `ERROR_ACCESS_DENIED (5)` counts as alive (process exists, not ours).
   Verified: own PID → alive; System PID 4 → access-denied → alive;
   PID 0 / 1 / 999999 → `ERROR_INVALID_PARAMETER (87)` → dead.
2. **`_pid_alive()`** routes `os.name == "nt"` to that probe instead of
   returning `False`. POSIX branch unchanged.
3. **`plugin.yaml`**: `platforms: [linux, macos, windows]`, version
   `1.4.2-win.1`.

Nothing else changed. `sqlite3` read-only URIs (`file:<path>?mode=ro`) work with
Windows backslash paths — measured on a copy of a live `state.db` (734 rows read).

## Verify

```bash
hermes plugins validate <this-dir>     # → Validation passed
hermes -p <trial-profile> plugins doctor office-3d
```

## Licence

MIT — original `LICENSE` kept unchanged. This file records the local
modifications as MIT requires the copyright/notice to travel with the code.
Re-sync from upstream with `git diff` against `origin/main` before any pull; the
patch is ~40 lines and only touches the two `_pid_alive` sites plus the manifest.
