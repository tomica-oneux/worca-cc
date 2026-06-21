# W-077: Docker Runtime Support

**Status:** Draft
**Priority:** P2
**Area:** cc
**Date:** 2026-06-19
**Depends on:** None

## Problem

Running worca on Windows via WSL2 is slow because projects on NTFS (`/mnt/c`) incur a filesystem
performance penalty through the 9P protocol bridge. This forces Windows developers to either accept
degraded performance or maintain a parallel Linux environment.

Docker with named volumes (stored on WSL2's ext4 filesystem) provides Linux-native filesystem speed
while keeping the developer on Windows — bind-mounts from NTFS are equally slow, but cloning into a
Docker named volume bypasses 9P entirely.

**Current state:** No Docker awareness exists anywhere in the codebase. The only runtime is local
execution (`src/worca/cli/run_pipeline.py:49` → `cmd_run()` selects between `run_pipeline.py` and
`run_worktree.py` at `:70`, then spawns via `subprocess.run()` at `:100` — no alternative runtime
path). The settings schema (`src/worca/settings.json:127` → `worca` key) has no `runtime` or
`docker` namespace. Environment variable filtering (`src/worca/utils/env.py:38` →
`RESERVED_PREFIXES`) strips any `WORCA_*` var, ruling out env-based Docker detection.

## Proposal

Add Docker as an optional, transparent runtime — the user sets `worca.runtime: "docker"` in
`settings.local.json` and `worca run` automatically delegates to a Docker container. The project is
cloned from its git remote into a named volume (ext4), the pipeline runs at native Linux speed, and
results are pushed back.

The existing local runtime stays unchanged as the default. This is purely additive with no breaking
changes.

**Design principle alignment:**
- Config namespaced under `worca` key — `worca.runtime`, `worca.docker.*`
- Secrets isolated to `settings.local.json` — runtime is per-machine, never committed
- Zero external Python deps — Docker utilities use stdlib `subprocess` + `shutil`
- Git worktrees as isolation boundary — Docker complements worktrees (they run inside the container)
- Tiered resolution — runtime follows the existing settings merge chain

## Design

### 1. Docker Detection (avoiding WORCA_* prefix)

- **Current state:** `src/worca/utils/env.py:38` defines `RESERVED_PREFIXES` including `WORCA_*` — any
  env var matching these prefixes is stripped with a warning.
- **Obstacle:** Using `WORCA_IN_DOCKER=1` for detection would conflict with the reserved prefix.
- **Resolution:** Detect Docker via `/.dockerenv` file (standard Docker marker), no env var needed.
  Fallback to checking `/proc/1/cgroup` for containerization hints on non-Docker runtimes (Podman, etc.).

```python
# src/worca/utils/docker.py
def is_in_docker() -> bool:
    """Detect if running inside a Docker container."""
    if Path("/.dockerenv").exists():
        return True
    # Fallback: check cgroup for container hints
    try:
        cgroup = Path("/proc/1/cgroup").read_text()
        return "docker" in cgroup or "containerd" in cgroup
    except (FileNotFoundError, PermissionError):
        return False
```

### 2. Docker Image Foundation

- **Current state:** No Docker infrastructure exists in the repo — no `Dockerfile`, no image build
  tooling, no container awareness.
- **Obstacle:** The image must ship Python 3.12 + Node.js 22 + git + gh CLI + worca itself (both the
  Python package and the built `worca-ui` bundle), but must not bake in project-specific dependencies
  (Java, Go, Rust, etc.) that bloat the base.
- **Resolution:** Multi-stage build with a user-extensible pattern — the base image covers the
  common toolchain; users extend it via `worca.docker.dockerfile` for project-specific tools.

Multi-stage, user-extensible base image at `docker/Dockerfile`:

```dockerfile
# Stage 1: Base tools
FROM ubuntu:24.04 AS base
RUN apt-get update && apt-get install -y \
    python3.12 python3.12-venv python3-pip \
    nodejs npm git curl \
    && npm install -g n && n 22 \
    && npm install -g @anthropic-ai/claude-code

# Stage 2: Worca layer
FROM base AS worca
WORKDIR /worca-cc
COPY . .
RUN pip install -e . \
    && cd worca-ui && npm ci && npm run build

WORKDIR /workspace
EXPOSE 3400
COPY docker/entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

**User-extensible pattern:** For projects needing Java, Go, Rust, etc., users extend the base:

```dockerfile
# my-project/Dockerfile.worca
FROM worca-cc:local
RUN apt-get update && apt-get install -y openjdk-17-jdk maven
```

Then set `worca.docker.dockerfile: "./Dockerfile.worca"` in `settings.local.json`.

### 3. Container Entrypoint Modes

- **Current state:** Pipeline entry points (`src/worca/scripts/run_pipeline.py`,
  `src/worca/scripts/run_worktree.py`) assume they run on the host with direct filesystem access to
  the project.
- **Obstacle:** Inside Docker, the project must be cloned from a git remote into the named volume
  (ext4) — there is no bind-mounted project directory.
- **Resolution:** A shell entrypoint that clones-or-updates the project before delegating to the
  existing pipeline scripts. Reuses existing clone on re-run (`git fetch` + checkout instead of
  fresh clone).

`docker/entrypoint.sh` handles three modes:

| Mode | Command | Behavior |
|------|---------|----------|
| `pipeline` | Default | Clone project → run worca pipeline → push results |
| `shell` | `--shell` | Clone + drop into bash |
| `ui` | `--ui` | Start worca-ui on 0.0.0.0:3400 |

```bash
#!/bin/bash
set -euo pipefail

MODE="${1:-pipeline}"
GIT_REMOTE="${WORCA_GIT_REMOTE:?Required}"
GIT_BRANCH="${WORCA_GIT_BRANCH:-main}"
PROJECT_DIR="/workspace/project"

# Clone or update repository
if [ -d "$PROJECT_DIR/.git" ]; then
    cd "$PROJECT_DIR"
    git fetch origin
    git checkout "$GIT_BRANCH"
    git reset --hard "origin/$GIT_BRANCH"
else
    git clone "$GIT_REMOTE" "$PROJECT_DIR"
    cd "$PROJECT_DIR"
    git checkout "$GIT_BRANCH"
fi

# Configure git credentials
if [ -n "${GH_TOKEN:-}" ]; then
    git config --global credential.helper "!f() { echo username=x-access-token; echo password=$GH_TOKEN; }; f"
fi

case "$MODE" in
    pipeline)
        exec python -m worca.scripts.run_pipeline "$@"
        ;;
    shell)
        exec /bin/bash
        ;;
    ui)
        exec node /worca-cc/worca-ui/server/index.js --host 0.0.0.0
        ;;
esac
```

**Note:** The entrypoint does NOT install project dependencies — the base image ships common tools
(Python, Node, git); the pipeline's agents handle project-specific setup commands (`npm install`,
`pip install`, `mvn compile`, etc.).

### 4. Docker Utilities Module

- **Current state:** No Docker awareness in `src/worca/utils/` — no container management, no image
  inspection, no daemon detection.
- **Obstacle:** Need container management without adding external dependencies (`docker-py`, etc.).
  The project's zero-external-dep constraint for utilities (stdlib `subprocess` + `shutil`) must be
  preserved.
- **Resolution:** Call the `docker` CLI via `subprocess.run()` — the user must have Docker
  installed, so the CLI is already a host prerequisite.

New file `src/worca/utils/docker.py` — zero external deps, pure stdlib:

```python
"""Docker detection and container management utilities."""
import shutil
import subprocess
from dataclasses import dataclass
from pathlib import Path

@dataclass
class DockerStatus:
    available: bool
    message: str | None = None

def is_docker_available() -> DockerStatus:
    """Check if Docker CLI is available and daemon is running."""
    if not shutil.which("docker"):
        return DockerStatus(False, "docker not found on PATH")
    try:
        result = subprocess.run(
            ["docker", "info"],
            capture_output=True,
            timeout=10,
        )
        if result.returncode != 0:
            return DockerStatus(False, "Docker daemon not running")
        return DockerStatus(True)
    except subprocess.TimeoutExpired:
        return DockerStatus(False, "Docker daemon not responding")

def image_exists(name: str) -> bool:
    """Check if a Docker image exists locally."""
    result = subprocess.run(
        ["docker", "image", "inspect", name],
        capture_output=True,
    )
    return result.returncode == 0

def build_image(context_path: str, tag: str, dockerfile: str | None = None) -> subprocess.CompletedProcess:
    """Build a Docker image."""
    cmd = ["docker", "build", "-t", tag]
    if dockerfile:
        cmd.extend(["-f", dockerfile])
    cmd.append(context_path)
    return subprocess.run(cmd, check=True)

def run_container(
    image: str,
    *,
    git_remote: str,
    branch: str,
    prompt: str,
    env_vars: dict[str, str],
    volumes: list[str],
    ports: list[str],
    mode: str = "pipeline",
) -> subprocess.CompletedProcess:
    """Run a worca container."""
    cmd = ["docker", "run", "--rm"]
    
    # Environment variables
    cmd.extend(["-e", f"WORCA_GIT_REMOTE={git_remote}"])
    cmd.extend(["-e", f"WORCA_GIT_BRANCH={branch}"])
    for key, value in env_vars.items():
        cmd.extend(["-e", f"{key}={value}"])
    
    # Volumes
    for vol in volumes:
        cmd.extend(["-v", vol])
    
    # Ports
    for port in ports:
        cmd.extend(["-p", port])
    
    cmd.append(image)
    cmd.append(mode)
    
    if mode == "pipeline":
        cmd.extend(["--prompt", prompt])
    
    return subprocess.run(cmd)
```

### 5. CLI Subcommand

- **Current state:** `src/worca/cli/main.py:234-258` registers subcommands (templates, cleanup,
  workspace, graphify, crg, models, pr) via `register_subcommand(sub)` calls. Dispatch at `:276-325`
  routes to each handler.
- **Obstacle:** None — the pattern is well-established.
- **Resolution:** Follow the identical pattern from `cleanup.py:553` and `templates.py:180`.

New file `src/worca/cli/docker_cmd.py` following the `register_subcommand()` pattern used by
`cleanup.py:553`, `templates.py:180`:

```python
"""worca docker — Docker image and container management."""
import argparse

def register_subcommand(sub: argparse._SubParsersAction) -> None:
    docker = sub.add_parser("docker", help="Docker runtime management")
    docker_sub = docker.add_subparsers(dest="docker_cmd")
    
    # worca docker build
    build = docker_sub.add_parser("build", help="Build worca Docker image")
    build.add_argument("--source", default=".", help="Path to worca-cc source")
    build.add_argument("--tag", default="worca-cc:local", help="Image tag")
    
    # worca docker status
    docker_sub.add_parser("status", help="Show Docker status")
    
    # worca docker shell
    shell = docker_sub.add_parser("shell", help="Start interactive shell in container")
    shell.add_argument("--project", help="Git remote URL of project")
    
    # worca docker prune
    docker_sub.add_parser("prune", help="Remove dangling images and volumes")

def cmd_docker(args: argparse.Namespace) -> int:
    from worca.utils.docker import is_docker_available, build_image, image_exists
    
    if args.docker_cmd == "build":
        return _cmd_build(args)
    elif args.docker_cmd == "status":
        return _cmd_status()
    elif args.docker_cmd == "shell":
        return _cmd_shell(args)
    elif args.docker_cmd == "prune":
        return _cmd_prune()
    else:
        print("Usage: worca docker <build|status|shell|prune>")
        return 1
```

Modify `src/worca/cli/main.py:234-250` to register the subcommand and `:297-318` to dispatch.

### 6. Transparent Runtime Switch

- **Current state:** `src/worca/cli/run_pipeline.py:49` — `cmd_run()` selects between
  `run_pipeline.py` and `run_worktree.py` at `:70` and spawns via `subprocess.run()` at `:100`.
  There is no runtime dispatch layer — execution always runs locally.
- **Obstacle:** The Docker path must intercept before the local script selection, but must not
  recurse when already running inside the container.
- **Resolution:** Add a ~10-line guard at the top of `cmd_run()` that checks `worca.runtime` and
  `is_in_docker()` before falling through to the existing local path.

Modify `src/worca/cli/run_pipeline.py:49`:

```python
# BEFORE (current code, lines 49-70):
def cmd_run(args: Namespace) -> None:
    git_root = _find_git_root()
    # ... arg validation ...
    scripts_dir = git_root / ".claude" / "worca" / "scripts"
    pipeline_script = scripts_dir / "run_pipeline.py"
    worktree_script = scripts_dir / "run_worktree.py"
    use_worktree = args.worktree
    # ... worktree fallback ...
    script: Path = worktree_script if use_worktree else pipeline_script
    cmd = [sys.executable, str(script)]
    # ... forward args ...
    result = subprocess.run(cmd, cwd=str(git_root))
```

```python
# AFTER (with runtime dispatch inserted):
def cmd_run(args: Namespace) -> None:
    git_root = _find_git_root()
    
    # --- Runtime dispatch (unchanged below if runtime == "local") ---
    settings = _load_runtime_settings(git_root)
    runtime = settings.get("worca", {}).get("runtime", "local")
    
    if runtime == "docker":
        from worca.utils.docker import is_in_docker
        if not is_in_docker():
            from worca.cli._docker_run import docker_run
            return docker_run(args, git_root, settings)
    
    # --- Existing local execution path (unchanged) ---
    scripts_dir = git_root / ".claude" / "worca" / "scripts"
    pipeline_script = scripts_dir / "run_pipeline.py"
    # ... rest unchanged ...
```

New file `src/worca/cli/_docker_run.py` — separated to keep imports clean:

```python
"""Docker delegation for worca run."""
import os
from pathlib import Path

def docker_run(args, git_root: Path, settings: dict) -> int:
    from worca.utils.docker import is_docker_available, image_exists, run_container
    
    # Guard checks
    status = is_docker_available()
    if not status.available:
        print(f"Error: {status.message}")
        return 1
    
    docker_cfg = settings.get("worca", {}).get("docker", {})
    image = docker_cfg.get("image", "worca-cc:local")
    
    if not image_exists(image):
        print(f"Error: Docker image '{image}' not found. Run 'worca docker build' first.")
        return 1
    
    # Detect git remote and branch
    git_remote = _get_git_remote(git_root)
    git_branch = _get_current_branch(git_root)
    
    # Collect environment variables
    env_vars = {
        "ANTHROPIC_API_KEY": os.environ.get("ANTHROPIC_API_KEY", ""),
    }
    if gh_token := os.environ.get("GH_TOKEN"):
        env_vars["GH_TOKEN"] = gh_token
    
    # Build volume mounts
    volumes = ["worca-workspace:/workspace", "worca-state:/root/.worca"]
    
    if docker_cfg.get("forward_ssh_agent", True) and os.environ.get("SSH_AUTH_SOCK"):
        ssh_sock = os.environ["SSH_AUTH_SOCK"]
        volumes.append(f"{ssh_sock}:{ssh_sock}")
        env_vars["SSH_AUTH_SOCK"] = ssh_sock
    
    if docker_cfg.get("forward_claude_auth", True):
        claude_dir = Path.home() / ".claude"
        if claude_dir.exists():
            volumes.append(f"{claude_dir}:/root/.claude:ro")
    
    # Run container
    result = run_container(
        image,
        git_remote=git_remote,
        branch=git_branch,
        prompt=args.prompt,
        env_vars=env_vars,
        volumes=volumes,
        ports=[f"{docker_cfg.get('ui_port', 3400)}:3400"],
        mode="pipeline",
    )
    
    return result.returncode
```

### 7. Configuration Schema

- **Current state:** `src/worca/settings.json:127-389` defines the `worca` namespace with stages,
  agents, models, loops, pricing, governance, etc. Settings are loaded via
  `src/worca/utils/settings.py:79` (`load_settings()`) with `.local.json` deep-merge at `:23-33`
  (`deep_merge()`). Worktree propagation at `src/worca/utils/runtime.py:14`
  (`PROPAGATED_LOCAL_WORCA_KEYS`) controls which keys copy to child worktrees.
- **Obstacle:** None — the existing merge chain handles new keys naturally via defaults.
- **Resolution:** Add `runtime` and `docker` under the `worca` key with safe defaults.

Add to `src/worca/settings.json` under the `worca` key (after the existing `code_review_graph`
block, around line 389):

```json
{
  "worca": {
    "runtime": "local",
    "docker": {
      "image": "worca-cc:local",
      "dockerfile": null,
      "ui_port": 3400,
      "forward_ssh_agent": true,
      "forward_claude_auth": true,
      "extra_env": []
    }
  }
}
```

Users set `worca.runtime: "docker"` in `.claude/settings.local.json` (per-machine, gitignored).

### 8. Credential Forwarding

- **Current state:** Secrets are handled via `settings.local.json` (gitignored) and propagated to
  worktrees by `src/worca/utils/runtime.py:31` (`propagate_runtime_local_keys()`). The reserved env
  key schema at `src/worca/schemas/reserved_env_keys.json` strips `WORCA_*` vars but does not
  restrict `ANTHROPIC_API_KEY`, `GH_TOKEN`, or `SSH_AUTH_SOCK`.
- **Obstacle:** Docker containers don't inherit host env vars or filesystem access by default.
- **Resolution:** Explicit forwarding via `-e` flags and volume mounts, configurable per key.

| Credential | Method | Config |
|------------|--------|--------|
| `ANTHROPIC_API_KEY` | `-e ANTHROPIC_API_KEY` env var | Required |
| `GH_TOKEN` | `-e GH_TOKEN` + git credential helper in entrypoint | Required for push/PR |
| SSH keys | Mount `SSH_AUTH_SOCK` as volume | `docker.forward_ssh_agent` (default: `true`) |
| Claude auth | Mount `~/.claude/` read-only | `docker.forward_claude_auth` (default: `true`) |

## Implementation Plan

### Phase 1: Docker Image Foundation
**Files:** `docker/Dockerfile`, `docker/entrypoint.sh`, `docker/docker-compose.yml`, `docker/.dockerignore`

**Tasks:**
1. Create `docker/Dockerfile` with multi-stage build (Ubuntu 24.04 + Python 3.12 + Node.js 22 + git + gh CLI + worca layer)
2. Create `docker/entrypoint.sh` with three modes (pipeline/shell/ui)
3. Create `docker/docker-compose.yml` convenience template with named volumes
4. Create `docker/.dockerignore` build context filter

### Phase 2: Docker Utilities Module
**Files:** `src/worca/utils/docker.py`, `tests/test_docker.py`

**Tasks:**
1. Implement `is_docker_available()`, `is_in_docker()`, `image_exists()` detection functions
2. Implement `build_image()`, `run_container()` subprocess wrappers
3. Write unit tests with mocked `subprocess.run`

### Phase 3: CLI Subcommand
**Files:** `src/worca/cli/docker_cmd.py`, `src/worca/cli/main.py`

**Tasks:**
1. Create `docker_cmd.py` with `register_subcommand()` and `cmd_docker()` following existing patterns at `src/worca/cli/cleanup.py:14`, `src/worca/cli/templates.py:12`
2. Register subcommand in `main.py:234-250`
3. Add dispatch case in `main.py:297-318`

### Phase 4: Transparent Runtime Switch
**Files:** `src/worca/cli/run_pipeline.py`, `src/worca/cli/_docker_run.py`

**Tasks:**
1. Add runtime check at top of `cmd_run()` (`run_pipeline.py:49`)
2. Create `_docker_run.py` with Docker delegation logic
3. Implement git remote/branch detection, env var collection, volume mounting

### Phase 5: Configuration
**Files:** `src/worca/settings.json`

**Tasks:**
1. Add `worca.runtime` and `worca.docker.*` defaults to settings schema

### Phase 6: Documentation
**Files:** `docs/docker-runtime.md`

**Tasks:**
1. Write setup guide with installation prerequisites
2. Document usage examples for all three modes
3. Document credential forwarding configuration
4. Document user-extensible image pattern (Java, Go, Rust examples)
5. Add troubleshooting section

## Considerations

- **Bind-mount from NTFS is equally slow** — the speed gain comes from cloning into a Docker named
  volume (ext4). This means the project must have a git remote. Acceptable because worca pipelines
  always push branches/create PRs.

- **Docker detection via `/.dockerenv`** — standard Docker marker file, no env var needed. Avoids the
  `WORCA_*` reserved prefix entirely (`src/worca/utils/env.py:40`). Falls back to checking
  `/proc/1/cgroup` for containerization hints on non-Docker runtimes (Podman, etc.).

- **Project dependencies** — the base image ships Python + Node + git (covers most web projects). For
  Java/Go/Rust/etc., users extend via `worca.docker.dockerfile`. The pipeline's agents handle
  project-level setup commands (`npm install`, `mvn compile`, etc.) — the image provides the tools,
  agents run the commands.

- **Docker Desktop on Windows uses WSL2 backend** — named volumes are stored in WSL2's ext4 filesystem,
  which is why they're fast. This is the core mechanism.

- **Cross-platform behavior.** The Docker runtime is functional on all three supported platforms, but
  the performance benefit varies:

  | Platform | Docker backend | Named volume FS | Speed benefit vs local | Notes |
  |----------|---------------|-----------------|----------------------|-------|
  | **Windows** | WSL2 | ext4 | **Large** — bypasses 9P/NTFS penalty | Primary motivation for this feature |
  | **macOS** | Linux VM (Apple Hypervisor / VZ) | ext4 | **Moderate** — avoids VirtioFS/gRPC-FUSE overhead on large trees | Useful for large monorepos |
  | **Linux** | Native | ext4/overlay2 | **None** — local FS is already native | Docker adds container overhead; use local runtime |

  All code paths are cross-platform: `subprocess.run(["docker", ...])`, `shutil.which("docker")`,
  and `Path("/.dockerenv").exists()` work on all three OSes. The `entrypoint.sh` runs inside the
  Linux container regardless of host OS.

  **Platform-specific caveat — SSH agent forwarding:** `SSH_AUTH_SOCK` volume mounting works natively
  on Linux/macOS. On Windows, Docker Desktop provides SSH agent sharing through its own mechanism —
  the `-e SSH_AUTH_SOCK` + volume mount pattern in `_docker_run.py` handles this, but users must
  enable "Use the WSL 2 based engine" in Docker Desktop settings (default since Docker Desktop 4.x).

- **Breaking changes:** None — Docker is purely additive, opt-in, default is `"local"`.

- **Migration:** None required — new config keys with defaults.

## Test Plan

### Unit Tests (`tests/test_docker.py`)

| Test | Validates |
|------|-----------|
| `test_is_docker_available_found` | Returns `True` when docker CLI found + daemon running |
| `test_is_docker_available_not_found` | Returns `(False, message)` when docker not on PATH |
| `test_is_docker_available_daemon_not_running` | Returns `(False, message)` when daemon offline |
| `test_is_in_docker_with_dockerenv` | Returns `True` when `/.dockerenv` exists |
| `test_is_in_docker_with_cgroup` | Falls back to `/proc/1/cgroup` check |
| `test_is_in_docker_negative` | Returns `False` when no container markers present |
| `test_image_exists_positive` | Checks `docker image inspect` returns 0 |
| `test_image_exists_negative` | Checks `docker image inspect` returns non-zero |
| `test_build_image_basic` | Constructs correct `docker build` command |
| `test_build_image_custom_dockerfile` | Includes `-f` flag when dockerfile specified |
| `test_run_container_pipeline_mode` | Constructs correct `docker run` with volumes/env/ports |
| `test_run_container_shell_mode` | Passes `shell` mode to entrypoint |
| `test_runtime_switch_local` | `cmd_run` takes local path when `runtime: "local"` |
| `test_runtime_switch_docker` | `cmd_run` delegates to Docker when `runtime: "docker"` |
| `test_runtime_switch_recursion_guard` | Falls through to local when `/.dockerenv` exists |

All tests mock `subprocess.run` — no actual Docker required.

### Integration / E2E Tests

No real Docker integration tests in CI — Docker is a host prerequisite that cannot be assumed in all
CI environments. The unit tests with mocked `subprocess.run` validate command construction and
control flow. Manual integration testing covers:

- Build the image locally (`worca docker build`), verify it produces a tagged image.
- Run `worca docker status` and confirm it reports the image + daemon state.
- Set `worca.runtime: "docker"` in `settings.local.json`, run `worca run --prompt "test"`, confirm
  the container starts and the pipeline executes inside it.
- Verify the recursion guard: inside the container, `worca run` falls through to local execution.

### Existing Tests to Update

None — this is a purely additive feature. Existing tests continue to use the local runtime.

### Acceptance Gate (done-criteria)

The feature is **done** when:
1. `test_runtime_switch_docker` passes — `cmd_run()` delegates to `docker_run()` when
   `runtime: "docker"` and not inside a container.
2. `test_runtime_switch_recursion_guard` passes — `cmd_run()` falls through to local execution when
   `/.dockerenv` exists (no infinite Docker-spawns-Docker loop).
3. `test_is_in_docker_with_dockerenv` passes — detection via `/.dockerenv` works.
4. `test_is_docker_available_found` and `test_is_docker_available_not_found` pass — daemon
   detection is correct in both cases.
5. `test_run_container_pipeline_mode` passes — the constructed `docker run` command includes correct
   volumes, env vars, ports, and mode.
6. `test_build_image_custom_dockerfile` passes — custom Dockerfile path is forwarded via `-f` flag.
7. All existing `pytest tests/` and `npx vitest run worca-ui/server/` suites remain green.

## Files to Create/Modify

### Files to Create

| File | Purpose |
|------|---------|
| `docker/Dockerfile` | Multi-stage image build |
| `docker/entrypoint.sh` | Container entrypoint (clone → run → push) |
| `docker/docker-compose.yml` | Convenience template with named volumes |
| `docker/.dockerignore` | Build context filter |
| `src/worca/utils/docker.py` | Docker detection + container management |
| `src/worca/cli/docker_cmd.py` | `worca docker` subcommand |
| `src/worca/cli/_docker_run.py` | Docker delegation for `worca run` |
| `docs/docker-runtime.md` | User documentation |
| `tests/test_docker.py` | Unit tests for docker utilities |

### Files to Modify

| File | Change |
|------|--------|
| `src/worca/cli/main.py` | Register `docker` subcommand + dispatch |
| `src/worca/cli/run_pipeline.py` | Add runtime check at top of `cmd_run()` |
| `src/worca/settings.json` | Add `worca.runtime` + `worca.docker.*` defaults |

## Out of Scope

- **Container registry publishing (GHCR/Docker Hub)** — local build only for now
- **CI Docker integration tests** — unit tests with mocked subprocess only
- **Dev Containers / `.devcontainer/`** — VS Code integration, future enhancement
- **Docker-in-Docker** — nested container execution
- **Fleet/workspace Docker orchestration** — runs inside a single container, same as local
