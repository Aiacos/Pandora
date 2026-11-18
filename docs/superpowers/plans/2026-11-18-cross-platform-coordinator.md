# Cross-platform Coordinator in Docker — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The Pandora Coordinator runs headless on Windows, Linux and macOS and in a container, with a transport seam, path mapping and safe command parsing, wire-compatible with v1.1.0.11 slaves and workstations.

**Architecture:** Four small flat modules next to `PandoraCoordinator.py` (`PandoraPaths`, `PandoraPathMap`, `PandoraProtocol`, `PandoraTransport`); the Coordinator is edited only where it touches paths, parsing or command files. Scheduling logic is untouched.

**Tech Stack:** Python standard library only at runtime; pytest for tests; Docker + Compose; GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-11-18-pandora-modernization-design.md` (branch `design/modernization`)

## Global Constraints

- Work on branch `feature/cross-platform-coordinator`, cut from `development`. Spec and plan stay on `design/modernization` and never enter the PR.
- Python 3.7-compatible syntax in `Pandora/Scripts/` (no walrus, no positional-only params, no `match`, no `list[str]` generics, no f-string `=`). Tests may assume 3.9+.
- No third-party imports at runtime. `psutil` and `winreg` only lazily, only on Windows.
- Upstream style: 4-space indent, `camelCase` functions, the GPL header block copied from `PandoraCoordinator.py` at the top of each new module.
- Wire format unchanged: folder layout, file names, list-literal command text.
- With no `PANDORA_*` variable set and no `PathMapping.json`, behaviour on Windows equals v1.1.0.11.
- Commits authored by the repo's git identity; no AI attribution trailers.
- All test commands run from the repo root: `python -m pytest tests -q`.

## Review Focus

1. A command file is half-written when the Coordinator reads it (sync services, SMB): expected — skipped this cycle, not deleted, no crash. → Task 3/5 test.
2. `PathMapping.json` is malformed JSON: expected — warning logged, no mapping, Coordinator keeps running. → Task 2 test.
3. Root folder not writable by the container user (wrong `PUID`): expected — clear message on stderr and non-zero exit, not a silent loop. → Task 5 test.
4. Windows path with mixed case / trailing separators / UNC (`\\NAS\Pandora\Jobs\`) mapped on Linux: expected — maps correctly. → Task 2 test.
5. `command` setting contains valid-looking but non-allow-listed call (`self.setConfig(...)`, `self.__class__`): expected — dropped with a warning, setting cleared. → Task 3 test.

---

### Task 1: `PandoraPaths` and test scaffolding

**Files:**
- Create: `Pandora/Scripts/PandoraPaths.py`, `tests/conftest.py`, `tests/test_paths.py`, `requirements-dev.txt` (`pytest`)

**Interfaces — Produces:**
- `getPlatform() -> str` — `"windows"`, `"linux"` or `"darwin"`.
- `getUserConfigDir(platform=None, env=None) -> str` — windows: `<USERPROFILE>/Documents/Pandora`; linux: `<XDG_CONFIG_HOME or ~/.config>/Pandora`; darwin: `~/Library/Application Support/Pandora`.
- `getUserConfigPath(platform=None, env=None) -> str` — `env["PANDORA_CONFIG"]` if set, else `<userConfigDir>/Pandora.json`.
- `getEnvOverrides(env=None) -> dict` — keys present only when the variable is set: `"rootPath"` (`PANDORA_ROOT`), `"repositoryPath"` (`PANDORA_REPOSITORY`), `"localMode"` (`PANDORA_LOCAL_MODE`; `1/true/yes/on` → `True`, `0/false/no/off` → `False`, case-insensitive; anything else raises `ValueError`).
- `conftest.py` inserts `Pandora/Scripts` in `sys.path` and provides fixture `fakeRoot(tmp_path)` returning a root with `Scripts/PandoraCoordinator`, `Slaves`, `Workstations` created.

- [ ] Tests: one per platform for `getUserConfigDir` with a fake `env`; `PANDORA_CONFIG` wins; `getEnvOverrides({})=={}`; `{"PANDORA_LOCAL_MODE":"False"}` → `{"localMode": False}`; `"maybe"` raises.
- [ ] Run, see them fail; implement; run, pass.
- [ ] Commit: `Add PandoraPaths: per-OS config locations and env overrides`

### Task 2: `PandoraPathMap`

**Files:** Create `Pandora/Scripts/PandoraPathMap.py`, `tests/test_pathmap.py`

**Interfaces — Produces:**
- `loadMappings(paths) -> list` — `paths` is an ordered list of JSON files; missing files skipped; returns concatenated `"mappings"` lists (each a dict with any of `windows`/`linux`/`darwin`). Malformed file: skipped, and the error string appended to module-level list returned by `getLoadErrors()`.
- `mapPath(path, mappings, targetOs=None) -> str` — longest-prefix match over every OS column except the target's own being a no-op; `\`≡`/`; windows column compared case-insensitively; prefix must end on a separator boundary (`P:/proj` does not match `P:/projects`); result uses `\` for windows, `/` otherwise; no match or empty input → input unchanged.
- `getMappingFiles(coordBasePath, userConfigDir) -> list` — `[<userConfigDir>/PathMapping.json, <coordBasePath>/PathMapping.json]` (per-machine first).

- [ ] Tests: `P:\show\a.hip`→linux `/mnt/projects/show/a.hip`; linux→windows gives backslashes; darwin→linux; `\\NAS\Pandora\Jobs\` → `/pandora/Jobs/`; `p:/SHOW` matches `P:/`; boundary case above; longest prefix wins over shorter; no rule → unchanged; malformed JSON file → `[]` and `getLoadErrors()` non-empty; per-machine file rules come first.
- [ ] Fail, implement, pass.
- [ ] Commit: `Add PandoraPathMap: translate paths between operating systems`

### Task 3: `PandoraProtocol`

**Files:** Create `Pandora/Scripts/PandoraProtocol.py`, `tests/test_protocol.py`

**Interfaces — Produces:**
- `parseCommand(text) -> list or None` — `ast.literal_eval`; returns the value only if it is a non-empty `list`; any exception or other type → `None`.
- `parseCall(text) -> (name, args, kwargs) or None` — accepts exactly one expression of the form `self.<identifier>(<literals>)` (optional trailing whitespace/`;`); args and keyword values must pass `ast.literal_eval`; anything else → `None`.
- `COORDINATOR_COMMANDS = ("searchUncollectedRnd", "collectOutput", "notifyWorkstations", "notifySlaves", "restartGDrive")`.
- `runCall(obj, text, allowed) -> bool` — parses, checks `name in allowed` and not starting with `_`, calls `getattr(obj, name)(*args, **kwargs)`; returns whether it ran.

- [ ] Tests: `'["taskUpdate", "abc", "t1", "finished", 1.5, 2, 3, 4]'` parses; `'__import__("os").system("x")'` → `None`; truncated `'["taskUpd'` → `None`; `'"str"'` → `None`; `parseCall("self.searchUncollectedRnd()")` → `("searchUncollectedRnd", [], {})`; `parseCall("self.collectOutput(jobCode='a')")` kwargs; rejects `self.a.b()`, `os.system('x')`, `self.f(open('x'))`, `self.f(); self.g()`, `self.__class__`; `runCall` returns `False` for `self.setConfig('a','b',1)` and never calls it.
- [ ] Fail, implement, pass.
- [ ] Commit: `Add PandoraProtocol: parse command files without eval/exec`

### Task 4: `PandoraTransport`

**Files:** Create `Pandora/Scripts/PandoraTransport.py`, `tests/test_transport.py`

**Interfaces — Produces:**
- `class Transport(object)` with methods raising `NotImplementedError`: `listWorkstationCommands(wsName) -> list`, `listSlaveCommands(slaveName) -> list`, `readCommand(handle) -> str`, `removeCommand(handle)`, `sendToSlave(slaveName, cmd) -> handle`. A handle is opaque to callers.
- `class FileTransport(Transport)`, `__init__(self, rootPath)`. Handles are absolute file paths.
  - `listWorkstationCommands`: sorted `Workstations/WS_<ws>/Commands/handlerOut_*`; missing dir → `[]`.
  - `listSlaveCommands`: sorted `Slaves/S_<slave>/Communication/slaveOut_*`; missing dir → `[]`.
  - `sendToSlave`: writes `str(cmd)` to `Slaves/S_<slave>/Communication/slaveIn_<NNNN>_<time.time()>.txt`, `NNNN` = highest existing numeric `slaveIn_` index + 1, zero-padded to 4, starting at `0001` (same algorithm as today's `sendCommand`); creates the folder if missing.
  - `removeCommand`: `os.remove`, raising on failure (caller logs).

- [ ] Tests: listing ignores `Pandora_update.zip` and non-prefixed files and is sorted; `sendToSlave("r1", ["renderTask","j","t"])` creates a file matching `^slaveIn_0001_\d+(\.\d+)?\.txt$` whose text equals `str([...])`; second send → `0002`; a non-numeric `slaveIn_x_1.txt` is ignored when numbering; missing folders → `[]`.
- [ ] Fail, implement, pass.
- [ ] Commit: `Add PandoraTransport: isolate command-file channels behind an interface`

### Task 5: Coordinator changes

**Files:** Modify `Pandora/Scripts/PandoraCoordinator.py`; create `tests/test_coordinator.py`

**Interfaces — Consumes:** everything from Tasks 1–4.
**Produces:** `PandoraCoordinator(start=True)`; `start=False` builds the object without looping. `setup() -> bool` (resolve paths, False if it cannot run), `run()` (the loop), `startCoordination()` unchanged in meaning.

Changes, in order — each is a line-level edit, nothing else moves:

- [ ] **Header (lines 36–54):** delete the `pyLibs`/`pVersion`/`sys.path.append`/`import psutil` block; import the four modules, `ast` not needed here, add `signal`.
- [ ] **Constructor split:** body of today's `__init__` up to the `while` loop becomes `setup()`; the loop plus the final log/notify becomes `run()`; `__init__(self, start=True)` sets defaults, calls `setup()` and, if it returned True and `start`, `run()` then `sys.exit()`. The `if __name__` block is unchanged.
- [ ] **Config resolution in `setup()`:** `pandoraConfig = PandoraPaths.getUserConfigPath()`; after reading `cData`, apply `PandoraPaths.getEnvOverrides()` on top (`rootPath` replaces both `globalRootpath` and `coordRootpath`; `repositoryPath`; `localMode`). When `PANDORA_ROOT` is set, a missing `Pandora.json` is not an error and `coordEnabled` is treated as True. After the shared-settings reads of `repository` and `localMode` (today lines ~188–201), re-apply the env overrides so environment wins.
- [ ] **Repository mapping:** the `repository` value from `Coordinator_Settings.json` and `repositoryPath` from the user config go through `PandoraPathMap.mapPath` with mappings from `getMappingFiles(self.coordBasePath, userConfigDir)` before `os.path.exists`. Load errors → `writeWarning`.
- [ ] **Writable check:** in `setup()`, after the root is known, try creating `coordBasePath`; on `OSError` print `"Pandora root is not writable: <path> (<error>)"` to stderr and return False; `__init__` then `sys.exit(1)`.
- [ ] **Transport:** `self.transport = PandoraTransport.FileTransport(self.slPath)`. `sendCommand` body becomes log + `self.transport.sendToSlave(slave, cmd)`. In `getJobAssignments` the `handlerOut_` loop iterates `self.transport.listWorkstationCommands(wsName)` (the `Pandora_update.zip` branch stays a direct file check before it). In `checkSlaves` the `slaveOut_` loop iterates `self.transport.listSlaveCommands(slaveName)`.
- [ ] **`handleCmd`:** read via `self.transport.readCommand`, parse via `PandoraProtocol.parseCommand`. If it returns `None` **and the file is younger than 60 seconds**, log level 1 and return without removing (half-written file); older than that, log level 3 and remove. Removal goes through `self.transport.removeCommand`.
- [ ] **`checkCommands`:** replace `exec(val)` with `PandoraProtocol.runCall(self, val, PandoraProtocol.COORDINATOR_COMMANDS)`; when it returns False, `writeWarning("Coordinator command rejected: %s" % val, 2)`.
- [ ] **Google Drive:** `getGDrivePath` returns `""` immediately unless `PandoraPaths.getPlatform() == "windows"`, and imports `winreg` (fall back to `_winreg`); fix the `"SOFTWARE\Google\Drive"` string to a raw string. `restartGDrive` imports `psutil` inside a `try` and logs "not available" on `ImportError` or non-Windows (today's `"psutil" not in locals()` guard always returns early — this makes the feature work again on Windows).
- [ ] **Signals/heartbeat/stdout:** `run()` installs `SIGTERM` and `SIGINT` handlers that set `self.close = True`; sleep in 1-second slices so shutdown takes ≤1 s after a cycle. Each cycle end touches `<coordBasePath>/heartbeat`. `writeLog` additionally prints the line to stdout when env `PANDORA_LOG_STDOUT` is truthy (set in the image; off by default so `pythonw` on Windows is unaffected).
- [ ] **Version string** → `"v1.2.0.0"`.

Tests (`tests/test_coordinator.py`), each using `fakeRoot`, `monkeypatch.setenv("PANDORA_ROOT", root)`, `PANDORA_CONFIG` pointing at a non-existent file, and `PandoraCoordinator(start=False)`:

- [ ] `test_setup_from_env`: `coordBasePath == <root>/Scripts/PandoraCoordinator`, `localMode is True` by default, `PANDORA_LOCAL_MODE=false` → False even if `Coordinator_Settings.json` says `localMode: true`.
- [ ] `test_cycle_runs_on_empty_root`: `startCoordination()` returns without raising; `Coordinator_Settings.json` exists.
- [ ] `test_workstation_command_is_handled_and_removed`: write `handlerOut_0001_1.txt` containing `["clearLog", "Coordinator"]`-style valid command for a command type that needs no job (use `["deleteWarning", "Coordinator", "x", 1]`); after a cycle the file is gone.
- [ ] `test_malicious_command_file_does_not_execute`: file content `__import__("pathlib").Path(r"<tmp>/pwned").touch()`, mtime set 120 s in the past; after a cycle `pwned` does not exist and the file is removed.
- [ ] `test_half_written_command_is_kept`: content `["taskUpd`, fresh mtime; after a cycle the file still exists.
- [ ] `test_command_setting_allow_list`: set `command` to `self.setConfig('settings','x',1)`; after a cycle `settings.x` is absent and `command == ""`.
- [ ] `test_job_is_assigned_to_active_slave`: build a job in `<root>/JobRepository/Jobs/<code>/PandoraJob.json` and slave folders by reading what `getJobAssignments`/`getAvailableSlaves`/`assignJobs` require (copy the minimal keys from the code; one slave with fresh `slaveActive_`, one with none); after a cycle exactly the active slave has a `slaveIn_0001_*.txt` whose parsed content starts with `"renderTask"`.
- [ ] `test_unwritable_root_exits_nonzero` (skip on Windows and when running as root): chmod root `0o500`, expect `SystemExit` with code 1.
- [ ] `test_run_stops_on_close_flag`: thread sets `coord.close = True`; `run()` returns within 5 s and `heartbeat` exists.
- [ ] Run full suite; pass. Also `python -m compileall -q Pandora/Scripts/Pandora{Paths,PathMap,Protocol,Transport,Coordinator}.py`.
- [ ] Commit: `Coordinator: run on Linux/macOS, safe parsing, env config, clean shutdown`

### Task 6: Container

**Files:** Create `docker/Dockerfile`, `docker/entrypoint.sh`, `docker-compose.yml`, `.env.example`, `.dockerignore`

- [ ] `Dockerfile`: `python:3.12-slim`; install `gosu`; copy only `Pandora/Scripts/Pandora{Coordinator,Paths,PathMap,Protocol,Transport}.py` to `/opt/pandora`; `ENV PANDORA_ROOT=/pandora PANDORA_LOCAL_MODE=true PANDORA_LOG_STDOUT=1 PYTHONUNBUFFERED=1`; `VOLUME /pandora`; `HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3` running a python one-liner that fails when `/pandora/Scripts/PandoraCoordinator/heartbeat` is missing or older than 120 s.
- [ ] `entrypoint.sh`: if started as root, create/reuse group and user for `PGID`/`PUID` (default 1000/1000) and `exec gosu` to it; otherwise run directly; `exec python /opt/pandora/PandoraCoordinator.py`.
- [ ] `docker-compose.yml`: service `coordinator`, `image: ghcr.io/aiacos/pandora-coordinator:latest`, `build: {context: ., dockerfile: docker/Dockerfile}`, `restart: unless-stopped`, `stop_grace_period: 30s`, `environment: PUID, PGID, TZ, PANDORA_LOCAL_MODE` from `.env` with defaults, `volumes: ["${PANDORA_ROOT_HOST:?set PANDORA_ROOT_HOST in .env}:/pandora"]`.
- [ ] `.env.example`: the four variables with comments.
- [ ] Verify locally: `PANDORA_ROOT_HOST=<tmp> docker compose up -d --build`; wait until `docker inspect` health is `healthy`; `Coordinator_Settings.json` owned by `PUID`; `docker compose stop` exits with code 0 in under 10 s; `docker compose down`.
- [ ] Commit: `Add Dockerfile and compose file for the Coordinator`

### Task 7: CI and documentation

**Files:** Create `.github/workflows/tests.yml`, `.github/workflows/docker.yml`, `Documentation/index/cross_platform.rst`; modify `Documentation/index.rst` (toctree), `README.md`, `.gitignore`

- [ ] `tests.yml`: on push and pull_request; matrix `os: [ubuntu-latest, macos-latest, windows-latest]` × `python: ["3.9", "3.13"]` plus ubuntu `3.11`; `pip install -r requirements-dev.txt`; `python -m pytest tests -q`.
- [ ] `docker.yml`: on pull_request build the image (no push) and run the Task 6 health check against a temp root; on push to the fork's default branch and tags, build `linux/amd64,linux/arm64` and push to `ghcr.io/${{ github.repository_owner }}/pandora-coordinator` (lower-cased), guarded by `if: github.repository_owner != 'RichardFrangenberg'` so it is inert upstream until the author opts in.
- [ ] `cross_platform.rst`: supported platforms table (Coordinator: Windows/Linux/macOS/Docker; everything else: Windows, others planned); config locations; `PANDORA_*` variables and precedence; `PathMapping.json` format with the two-rule example from the spec; Docker quick start (copy `.env.example`, `docker compose up -d`); the security note (Coordinator no longer executes code from command files; slaves unchanged).
- [ ] `README.md`: short "Coordinator in Docker" section linking to the page.
- [ ] `.gitignore`: `.env`, `.pytest_cache/`, `__pycache__/`.
- [ ] Commit: `Add CI, cross-platform documentation`

### Task 8: Fork, push, pull request

- [ ] `gh repo fork RichardFrangenberg/Pandora --remote=false`; add remote `fork`; push `feature/cross-platform-coordinator` and `design/modernization`.
- [ ] Confirm CI is green on the fork branch (all OS jobs + docker job); fix until it is.
- [ ] Open the PR against `RichardFrangenberg/Pandora:development`. Text in English, first person, covering: what and why; tested / not tested; compatibility; the one behaviour change; roadmap; offer to keep contributing / maintain the fork; Deadline 10 maintenance-mode context (stated accurately: maintenance mode since 7 Nov 2025, no EOL date); contact e-mail. No AI attribution.
