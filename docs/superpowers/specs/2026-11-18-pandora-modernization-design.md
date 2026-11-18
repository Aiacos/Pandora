# Pandora modernization — design

Date: 2026-11-18
Status: draft, awaiting approval
Scope of this document: the overall roadmap, and the full design of **sub-project 1** (cross-platform Coordinator in Docker). Sub-projects 2–5 are outlined only; each gets its own design before it is built.

## Intent and why

Pandora is an open-source render farm manager by Richard Frangenberg (Prism Pipeline). Its last commit is from January 2022 (v1.1.0.11). It is Windows-only, written for Python 2.7/3.7 with PySide/PySide2, and Prism 2 no longer ships a Pandora plugin.

AWS Thinkbox Deadline 10 entered maintenance mode on 7 November 2025: no new features, AWS steering users to Deadline Cloud, no end-of-life date announced. Small studios that want an on-premises farm manager integrated with Prism have fewer good options every year. Pandora is the natural candidate to fill that gap.

The owner of this work (Lorenzo, GitHub `Aiacos`) wants to run Pandora on a real mixed farm and give the work back upstream:

- render machines and workstations on **Windows, Linux and macOS**, some local, some remote over VPN;
- the **Coordinator on a NAS**, run with `docker compose`;
- submission from **Prism 2.1.4**, for every DCC Prism supports that can render on a farm.

## Goals

1. Pandora runs on Windows, Linux and macOS.
2. The Coordinator runs headless in a container on a NAS.
3. Jobs submitted on one OS render on another (path mapping).
4. Remote machines over VPN are first-class.
5. Prism 2.1.4 can submit to Pandora through its render farm plugin API.
6. Every step is offered upstream as a pull request to `RichardFrangenberg/Pandora` (`development` branch), from the fork `Aiacos/Pandora`. If the author resumes development, his repository is the home of the project; otherwise the fork is.

## Non-goals

- Rewriting Pandora as a new product. The code must stay recognisable to its author.
- A database or web UI.
- Cloud rendering, licence management, usage billing.
- Supporting Python 2.

## Anti-goals

- A pull request too large or too opinionated for the author to review. Each PR must be reviewable on its own and leave Pandora working.
- Breaking existing Windows installations. A Windows user who updates must keep their config, their jobs and their slaves.
- Breaking the wire format, so that a new Coordinator cannot talk to not-yet-updated slaves and workstations.

## Constraints

- Licence is GPL-3.0; all additions stay GPL-3.0.
- Upstream conventions: flat `Pandora/Scripts/Pandora*.py` modules, 4-space indent, existing naming (`camelCase` methods). New code follows them.
- Code that runs inside a DCC (submitters, later the Prism plugin) must run on the DCC's own Python. Supported floor: **Python 3.9** (VFX Reference Platform CY2022). The Coordinator container uses a current Python.
- Existing Windows installs launch the Coordinator with a bundled Python 3.7 (`PythonLibs/Python37/Pandora Coordinator.exe`, not in the repository). So in sub-project 1 the Coordinator and the new modules are written in **Python 3.7-compatible syntax** (no walrus, no positional-only parameters, no `match`, no built-in generics), even though CI can only test 3.9 and newer. The PR says 3.7 is expected to work but untested.
- Commits and PRs are authored by the owner's git identity (`uni.lorenzo.a@gmail.com`). No AI attribution lines in commits or PR text.

## Approaches

**A. Incremental port, file protocol unchanged.** Port to Python 3 and three OSes, keep the "everything is files in a shared folder" protocol exactly as is. Smallest diff, easiest for upstream, but the file protocol stays baked into every module.

**B. Rewrite as a network service.** Coordinator with HTTP API and a database, workers connect over the network. Most robust; but a new product, nothing mergeable upstream, months before the first render.

**C. Incremental port, with the transport isolated behind one interface (chosen).** As A, but every read/write of *messages* (commands, status updates) goes through one module. The only implementation shipped first is the existing file layout, byte-compatible. A network transport can be added later without touching scheduling logic.

The owner chose **C**: it modernises the code and makes Pandora more useful over VPN, while still producing upstream-sized PRs.

Note on VPN, so the plan is not oversold: with a VPN, remote machines can already mount the NAS share, so the file transport works across sites from day one. A network transport later buys lower latency and robustness on slow links, not reachability. That is why it is a later sub-project and not part of the first PR.

## Roadmap

| # | Sub-project | Outcome | Upstream PR |
|---|---|---|---|
| 1 | Cross-platform Coordinator in Docker | Coordinator runs on a NAS; transport seam, path mapping, platform paths, tests, CI | PR 1 |
| 2 | Slave on Linux and macOS | Render nodes on all three OSes; slave uses path mapping; headless slave mode; optional Syncthing for non-VPN machines | PR 2 |
| 3 | Prism 2.1.4 render farm plugin | `Prism/Plugins/Custom/Pandora`, modelled on the Deadline plugin (`registerRenderfarmPlugin`), for Houdini, Maya, 3ds Max, Blender | PR 3 (plugin may live in its own repo) |
| 4 | More DCCs | Nuke, Cinema 4D, After Effects, ComfyUI render launchers on the slave | PR 4 |
| 5 | GUIs on PySide6 | Render Handler, Settings, Tray on all three OSes; installer replaced by `pip`/scripts | PR 5 |
| later | Network transport | HTTP transport behind the seam from sub-project 1 | — |

Photoshop and PureRef are Prism apps but do not render on a farm; they are out of scope.

The Prism plugin is third because it needs a farm that already runs on Linux to submit to.

---

# Sub-project 1 — cross-platform Coordinator in Docker

## What exists today

`Pandora/Scripts/PandoraCoordinator.py` (2,400 lines, one class) is headless and does not import Qt or `PandoraCore`. Its constructor reads the user config, then loops forever: every `coordUpdateTime` seconds it runs `startCoordination()` (read commands, check slaves, assign jobs, notify), and finally calls `sys.exit()`.

All communication is files under one root folder:

- `<root>/Scripts/PandoraCoordinator/` — Coordinator settings, log, `ActiveSlaves.json`, `command.txt`, `EXIT.txt`
- `<root>/Slaves/S_<name>/Communication/slaveIn_NNNN_<time>.txt` — commands to a slave
- `<root>/Workstations/WS_<name>/…` — submissions and commands from workstations
- `<root>/JobRepository/{Jobs,ProjectAssets}` — job data (location overridable)

What blocks Linux and containers:

- config is read from `os.environ["userprofile"]\Documents\Pandora\Pandora.json`;
- `_winreg` and `googledrivesync.exe` handling (Google Drive restart feature);
- `psutil` imported unconditionally from a bundled `PythonLibs/Python37` folder that is not in the repository;
- Python 2 branches (`unicode`, `sys.version[0]`);
- the constructor never returns, so nothing can be tested;
- paths stored in jobs and settings are absolute Windows paths.

Two things are unsafe once the farm spans a VPN: command files are parsed with `eval()`, and the `command` setting is run with `exec()`. Anyone who can write a file into the shared folder can run code on the Coordinator.

## Design

### New modules (in `Pandora/Scripts/`, upstream naming)

**`PandoraPaths.py`** — where things live on each OS.
- User config directory: Windows keeps `%USERPROFILE%\Documents\Pandora` (existing installs untouched); Linux `$XDG_CONFIG_HOME/Pandora` (default `~/.config/Pandora`); macOS `~/Library/Application Support/Pandora`.
- Temp directory via `tempfile`, not `os.environ["temp"]`.
- Environment overrides, so a container needs no config file: `PANDORA_CONFIG` (path to `Pandora.json`), `PANDORA_ROOT`, `PANDORA_REPOSITORY`, `PANDORA_LOCAL_MODE`.
- Precedence, highest first: environment variable; the shared `Coordinator_Settings.json` (`repository`, `localMode` — today this already overrides the user config); the user's `Pandora.json`. When no environment variable is set, behaviour is exactly today's.
- Local mode: in `localMode` every machine reads and writes the same root directly; with it off, the Coordinator also collects outputs and pings slaves through a sync service. The owner's farm mounts the NAS share on every machine (remote ones over VPN), so the compose file defaults to `PANDORA_LOCAL_MODE=true`. Both modes stay supported and both are covered by the cycle tests.

**`PandoraPathMap.py`** — translate a path written on one OS into the same location on another.
- Rules live in one farm-wide file, `<root>/Scripts/PandoraCoordinator/PathMapping.json`, so every machine reads the same table:
  ```json
  {"mappings": [
    {"windows": "P:/", "linux": "/mnt/projects", "darwin": "/Volumes/projects"},
    {"windows": "//nas/pandora", "linux": "/pandora", "darwin": "/Volumes/pandora"}
  ]}
  ```
- `mapPath(path, targetOs=<this OS>)`: longest-prefix match against every OS column, case-insensitive for the Windows column, `\` and `/` treated alike, result in the target OS's separator. A path that matches no rule is returned unchanged. A missing or empty file means no mapping, which is today's behaviour.
- A per-machine override file in the user config directory is allowed (same format, checked first), because the Coordinator's container sees the root at `/pandora` regardless of how other Linux machines mount it.
- In this sub-project its only caller is the Coordinator's `repository` setting: that value is written by a Render Handler on some workstation (typically as a Windows or UNC path) and must resolve inside the container. The Coordinator reads no other absolute paths — job files carry names, counts and relative asset names, and output collection builds its paths from the root. The module still ships here, with its tests and documentation, because the mapping file format is farm-wide and the slave (sub-project 2) depends on it.

**`PandoraTransport.py`** — the seam.
- The seam covers exactly the three command-file channels, and nothing else:
  1. workstation → Coordinator: `Workstations/WS_<name>/Commands/handlerOut_*`;
  2. slave → Coordinator: the slave's outgoing command files in `Slaves/S_<name>/Communication/`;
  3. Coordinator → slave: `Slaves/S_<name>/Communication/slaveIn_NNNN_<time>.txt`.
- An abstract `Transport` with: list pending incoming commands (per channel), remove one once handled, send a command to a slave. There is no Coordinator → workstation command today, so there is no such operation.
- `FileTransport(rootPath)` implements them with **exactly** today's folder layout and file naming. A not-yet-updated slave or workstation cannot tell the difference. Files are written in the same encoding as today on Windows.
- **Outside the seam, unchanged and still plain file access:** slave liveness from the mtime of `slaveActive_<name>`; slave state read from `slaveSettings_<name>.json`; mirroring of logs, settings and job JSON into `Workstations/Logs/…` and `LogCache.json`; screenshot copies; `Pandora_update.zip` in the workstation `Commands` folder; scene files and rendered frames. Workstations also write `slaveIn_` files directly, bypassing the Coordinator; that stays as is. A future network transport will have to deal with these, and that is part of that later design, not this one.
- `PandoraCoordinator.sendCommand` and the two places that scan for incoming command files are rewritten to call the transport.

**`PandoraProtocol.py`** — parsing messages safely.
- Command files keep their current format (a Python list literal such as `["taskUpdate", jobCode, …]`), but are parsed with `ast.literal_eval`, never `eval`.
- The Coordinator's own `command` setting is run with `exec` today. In the Render Handler it is a free-text field whose only preset is `self.searchUncollectedRnd()`. It is replaced by a parser that accepts only the form `self.<name>(<literal args>)` and calls `<name>` only if it is on an explicit allow-list of Coordinator methods. Anything else is logged as a warning and dropped. Consequence: the preset keeps working; arbitrary Python typed into that field no longer runs.
- **Slave-bound command strings are not filtered in this sub-project.** Strings such as `self.shutdownPC(restart=True)`, `self.stopRender()`, `self.startTeamviewer()` are forwarded by the Coordinator verbatim, and the slave still `exec`s them and `eval`s its command files. Filtering them on the Coordinator would break free-typed slave commands without making slaves safe, since workstations can write to slaves directly. The slave gets the same parser in sub-project 2. The PR therefore claims only that the **Coordinator host** no longer executes code from the shared folder.

### Changes to `PandoraCoordinator.py`

- Split the constructor: `__init__` only sets state and resolves paths; `run()` holds the loop; `startCoordination()` stays one cycle. `if __name__ == "__main__"` calls `run()`. Behaviour when launched the old way is identical.
- Config and root resolution go through `PandoraPaths`.
- Remove Python 2 branches and the bundled-`PythonLibs` path insertion.
- `psutil`, `_winreg` and the Google Drive restart become Windows-only and optional: imported lazily, and the feature reports "not available" elsewhere instead of raising.
- Handle `SIGTERM`/`SIGINT` by finishing the current cycle and exiting cleanly (what `docker stop` sends). The existing `EXIT.txt` / `command.txt` mechanisms stay.
- Write a heartbeat file each cycle (`<root>/Scripts/PandoraCoordinator/heartbeat`), used by the container health check.
- Log lines also go to stdout so `docker logs` shows them; the log file is unchanged.
- Fix the stale version string (`v1.1.0.6` in the file vs `v1.1.0.11` released). The version number for this work is left to the author; the fork uses `v1.2.0.0`.

Everything else in the Coordinator — job assignment, priorities, slave checks, collecting output — is **not** restructured. Only the lines that touch paths, parsing or message files change.

### Container

- `docker/Dockerfile`: slim official Python image, no third-party Python dependencies at runtime, runs as a non-root user whose UID/GID come from `PUID`/`PGID` (NAS shares are picky about ownership). Built for `linux/amd64` and `linux/arm64`, since NAS CPUs are either.
- `docker-compose.yml` at the repository root:
  - service `coordinator`: the root folder bind-mounted at `/pandora`; `PANDORA_ROOT=/pandora`; `restart: unless-stopped`; health check = heartbeat file newer than a few cycles; `TZ` passed through.
  - `.env.example` documenting `PANDORA_ROOT_HOST`, `PUID`, `PGID`, `TZ`.
- Image published from the fork to GitHub Container Registry by CI, so the NAS can `docker compose pull` without building. Building locally stays possible.
- Syncthing is **not** in this sub-project (it belongs with remote slaves in sub-project 2).

### Tests and CI

- `pytest` suite under `tests/`, no DCC and no Qt required:
  - `PandoraPathMap`: mapping in all six OS directions, UNC paths, case, no-match, longest-prefix.
  - `PandoraPaths`: per-OS locations and environment overrides.
  - `PandoraProtocol`: valid commands parse; code such as `__import__("os").system(…)` is rejected; the allow-list holds.
  - `FileTransport`: files it writes match the legacy names; files written in the legacy format are read.
  - Coordinator cycle: build a fake root in a temp directory (one job, two slaves, one disabled), run `startCoordination()` once, assert the job is assigned to the right slave and the command file appears.
- GitHub Actions: tests on Ubuntu, macOS and Windows across Python 3.9–3.13; a job that builds the image and runs the container against a fake root until it is healthy. Upstream has no CI today; the workflow is included in the PR and is easy for the author to drop.

### Documentation

- `README.md`: a short "Coordinator in Docker" section and the supported-platform table.
- `Documentation/index/`: one new page on path mapping and the container, in the existing Sphinx structure.

## Compatibility

- Existing Windows install, updated: same config file, same folders, same command files. Path mapping file absent → no behaviour change.
- New Coordinator with old slaves/workstations: works (wire format unchanged).
- The one deliberate behaviour change: arbitrary code in command files or the `command` setting is no longer executed. This is called out in the PR.

## Pull request to upstream

- Fork `RichardFrangenberg/Pandora` to `Aiacos/Pandora`; branch `feature/cross-platform-coordinator` off `development`. This design document stays on a separate branch of the fork and is not part of the PR.
- A handful of commits that each stand alone (paths, path map, protocol, transport, Coordinator changes, Docker, tests/CI, docs).
- PR text in English, written in the owner's voice, containing: what changed and why; how it was tested and what was **not** tested; the compatibility notes above; the roadmap; the offer to keep contributing and to maintain the fork if the author prefers not to resume; the point that Deadline 10 is in maintenance mode and that this makes Pandora worth reviving; the owner's e-mail as contact.
- The PR text is shown to the owner before it is submitted.

## How this gets verified

- Test suite green on the three OSes in CI.
- On the development machine (Fedora, Docker 29): `docker compose up`, Coordinator healthy against a fake root, a hand-written job gets assigned to a simulated slave, `docker stop` exits cleanly.
- **Not verifiable here:** a real render through a DCC, a real NAS, Windows and macOS hosts beyond CI. The PR says so; the owner tests on the real farm.

## Calls made by the designer (not decided by the owner — correct any of these)

1. Python floor 3.9; Python 2 dropped.
2. New modules stay flat in `Pandora/Scripts/` with upstream naming, rather than becoming a pip package now.
3. Path mapping is one farm-wide JSON file next to the Coordinator settings, plus an optional per-machine override.
4. On the Coordinator, `eval`/`exec` are replaced by literal parsing and an allow-list, accepting that arbitrary Python typed into the Coordinator command field no longer runs. Slaves are left as they are until sub-project 2.
11. Coordinator and new modules stay Python 3.7-compatible in syntax so that existing Windows installs keep running, although only 3.9+ is tested.
12. The transport seam covers only the three command-file channels; status, heartbeat and log mirroring stay direct file access.
13. Compose defaults to local mode, on the assumption that remote machines mount the NAS share over the VPN.
5. Linux config under XDG, macOS under Application Support, Windows unchanged.
6. Image published to GHCR from the fork; multi-arch.
7. CI workflow included in the upstream PR.
8. Order of sub-projects as in the roadmap; Prism plugin third.
9. Photoshop and PureRef excluded.
10. The owner's e-mail appears in the PR text as contact, in addition to being the commit author address.

## Left to the builder

- Exact method names and signatures of `Transport`.
- The allow-list contents (derived from what the Render Handler actually sends).
- Base image tag, health-check timings, CI matrix details.
- How the Coordinator test fixtures are laid out.
