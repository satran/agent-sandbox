# agents

Run coding agents (Claude Code, Gemini CLI, …) inside throwaway-but-persistent
docker containers, one per agent and project, so they can run with permissions
disabled without touching the rest of your machine.

## Quick start

```sh
# put the script on your PATH (it resolves its own location, so a symlink is fine)
ln -s "$PWD/bin/sb" ~/.local/bin/sb

cd ~/src/myproject
sb claude       # build images if needed, start container, attach
# ... work ...  C-b d to detach
sb claude       # re-attach to the same session
```

Requirements on the host: `bash`, `docker`, `shasum`.

## Usage

```
usage: sb <agent> [path] [options] [-- agent-args...]
       sb list

options:
  --rebuild    rebuild the agent image (implies --recreate)
  --recreate   remove this project's container and start a fresh one
  --stop       stop this project's container
  -h, --help   show this help
```

- `path` defaults to the current directory.
- Anything after `--` is appended to the agent command, e.g.
  `sb claude -- --resume`. It only applies when the tmux session is first
  created; if the session is already running, the args are ignored with a warning.
- `sb list` shows all sandbox containers with their status and project path.

## How it works

For `sb <agent> <path>`:

1. **Config.** Sources `containers/<agent>/sandbox.conf` (see
   [Adding an agent](#adding-an-agent)).
2. **Path mapping.** The project is mounted at the same path relative to
   `$HOME`, so agent session history keyed by path stays stable:
   - `~/src/foo` → `/home/satran/src/foo`
   - `/opt/foo` (outside `$HOME`) → `/home/satran/ext/opt/foo`

   Mounting `$HOME` itself or `/` is refused.
3. **Images.** `sandbox-base` is built from `containers/base`, then
   `sandbox-<agent>` from `containers/<agent>`. Each is built only if missing,
   or always with `--rebuild` (the layer cache keeps that fast).
4. **Container.** It is named `sandbox-<agent>-<path-slug>-<6-char sha1 of path>`
   and labelled `sandbox.agent` / `sandbox.project`. It runs `sleep infinity`, so
   it stays alive between sessions; a stopped container is restarted. The mounts are:
   - the project directory (read-write)
   - each `CONFIG_MOUNTS` entry (host side under this repo, created if missing)
   - `~/.gitconfig` read-only, if present

   Environment: `PATH`, `COLORTERM=truecolor`, plus `AGENT_ENV`.
5. **Attach.** `docker exec` attaches to a tmux session named `sandbox`:
   - window 1 (named after the agent) runs `AGENT_CMD`, and drops to bash when the agent exits
   - window 2 (`shell`) is a plain shell in the project

## Layout

```
bin/sb                       the launcher
containers/base/             shared base image (sandbox-base)
containers/<agent>/          Dockerfile (FROM sandbox-base) + sandbox.conf
containers/docker-compose.yml  older compose-based setup (mounts ../../src)
home/<agent>/                persisted agent config & credentials (gitignored)
```

### The base image

`containers/base/Dockerfile` (Ubuntu 26.04) provides everything the agents share:
- the user `satran` with passwordless sudo
- curl, vim, emacs-nox, tmux, fish and git
- python3 with pip and venv (`python` → `python3`), plus node and npm
- `~/.local/bin` on `PATH`

Agent Dockerfiles only add the agent itself.

### Agents

| agent      | `sandbox.conf` | runnable with `bin/sb` |
|------------|----------------|-----------------------------|
| `claude`   | yes            | yes                         |
| `gemini`   | yes            | yes                         |
| `omp`      | no             | no, image only              |
| `opencode` | no             | no, image only              |
| `sandbox`  | no             | no, plain dev box           |
| `cliproxy` | —              | empty placeholder           |

## Adding an agent

1. Create `containers/<name>/Dockerfile`:

   ```dockerfile
   FROM sandbox-base

   # user-level install (lands in ~/.local/bin, already on PATH)
   RUN curl -fsSL https://example.com/install.sh | bash

   # or a system-wide install:
   # USER root
   # RUN npm install -g some-agent
   # USER satran
   ```

2. Create `containers/<name>/sandbox.conf`. It is bash, sourced by `bin/sb`:

   ```bash
   # host paths are relative to the repo root; created automatically
   CONFIG_MOUNTS=("home/<name>/.<name>:/home/satran/.<name>")
   AGENT_ENV=(SOME_VAR=value)
   AGENT_CMD=(<name> --some-flag)   # required
   ```

3. Run `sb <name> --rebuild`.

## Known issues

- The container user `satran` is hard-coded in `bin/sb` (`CUSER`) and in
  `containers/base/Dockerfile`.
- Security: the agents run with permission checks disabled and the container
  user has passwordless sudo, so the container is the only boundary. The
  project directory is mounted read-write, and the agent can change anything in it.
