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

These environment variables are read when a container is **created**, so change
them together with `--recreate`:

| variable | default | meaning |
|---|---|---|
| `SB_MEMORY` | unset | memory cap, e.g. `8g` |
| `SB_CPUS` | unset | CPU cap, e.g. `4` |
| `SB_PIDS` | `2048` | process limit |
| `SB_DNS` | `1.1.1.1 8.8.8.8` | resolvers; must be reachable, i.e. not on your LAN |
| `SB_BLOCKED_NETS` | see [Isolation](#isolation) | networks to deny egress to |
| `SB_FIREWALL` | `on` | `off` disables the egress firewall (debugging only) |

## How it works

For `sb <agent> <path>`:

1. **Config.** Sources `containers/<agent>/sandbox.conf` (see
   [Adding an agent](#adding-an-agent)).
2. **Path mapping.** The project is mounted at the same path relative to
   `$HOME`, so agent session history keyed by path stays stable:
   - `~/src/foo` → `/home/agent/src/foo`
   - `/opt/foo` (outside `$HOME`) → `/home/agent/ext/opt/foo`

   Mounting `$HOME` itself or `/` is refused.
3. **Images.** `sandbox-base` is built from `containers/base`, then
   `sandbox-<agent>` from `containers/<agent>`. Each is built only if missing,
   or always with `--rebuild` (the layer cache keeps that fast).
4. **Container.** It is named `sandbox-<agent>-<path-slug>-<6-char sha1 of path>`
   and labelled `sandbox.agent` / `sandbox.project`. It runs `sandbox-init`, which
   installs the egress firewall and then idles, so the container stays alive
   between sessions; a stopped container is restarted. The mounts are:
   - the project directory (read-write)
   - each `CONFIG_MOUNTS` entry (host side under this repo, created if missing)
   - a filtered copy of your global git config, read-only (see
     [Isolation](#isolation))
   - an empty read-only tmpfs over this repo's `home/`, if the project contains it

   Environment: `PATH`, `HOME`, `USER`, `COLORTERM=truecolor`, plus `AGENT_ENV`.
5. **Attach.** `docker exec --user agent` attaches to a tmux session named `sandbox`:
   - window 1 (named after the agent) runs `AGENT_CMD`, and drops to bash when the agent exits
   - window 2 (`shell`) is a plain shell in the project

   Before attaching and again after you detach, `sb` checks the project for
   changed host-executed files (see [Isolation](#isolation)).

## Isolation

The agent runs with its permission checks disabled, so the container is the
only thing standing between it and your machine:

- **No root.** There is no `sudo` and the `agent` user is not in any privileged
  group. Setuid and setgid bits are stripped from every file in the image, and
  `--security-opt no-new-privileges` stops a setuid binary or file capability
  from working even if one reappears.
- **No capabilities.** `--cap-drop ALL`. The container keeps only `NET_ADMIN`,
  and only so that `sandbox-init` — the root process the agent cannot reach —
  can install the firewall at startup.
- **No route to your network.** Egress is rejected to `10.0.0.0/8`,
  `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16` (which covers cloud
  instance metadata), `100.64.0.0/10` and `192.0.0.0/24`. That blocks your
  router, the docker host and anything you have listening on localhost or a LAN
  address, while leaving the public internet, DNS and `git`/`npm`/`pip`
  working. IPv6 is disabled by sysctl and dropped by ip6tables. Inbound is
  limited to established connections and the bridge, so published ports still
  work.
- **Its own network.** Sandboxes share a `sandbox` bridge with inter-container
  communication disabled, and use public resolvers rather than the host's, so
  one sandbox cannot reach another.
- **Resource limits.** 2048 processes, no core dumps, and optional memory and
  CPU caps, so a runaway agent cannot take the host down.
- **Nothing privileged is mounted.** No docker socket, no SSH agent, no host
  `$HOME`; mounting `$HOME`, `/` or anything under this repo's `home/` as the
  project is refused. If the project tree contains this repo, `home/` is
  shadowed with an empty tmpfs so the agent cannot read the other agents'
  credentials.
- **No host git secrets.** Your global git config is not mounted as is: `sb`
  writes a copy with only identity and harmless preferences (`user.name`,
  `user.email`, `alias.*`, `color.*`, `pull.*`, `push.default` and the like) to
  `home/.sb/<container>.gitconfig`. Tokens in `url.*.insteadOf`,
  `http.*.extraheader` or `credential.*` stay on the host. The copy is
  regenerated when the container is created, so use `--recreate` after
  changing your git config.
- **Warnings about host-executed files.** The agent can write files that the
  *host* later runs: `.git/hooks/*`, `.git/config` (`core.fsmonitor`,
  `core.hooksPath`, filter drivers), `.envrc` and `.vscode/*.json`. Your next
  `git status` or editor open would run them outside the sandbox. `sb` hashes
  these (in the project and nested repos up to 4 levels deep, skipping
  `node_modules` and `.venv`) when you attach and when you detach, and
  prints a warning listing any that changed. It only warns; nothing is
  blocked. Changes made while you are detached are reported at the next
  attach, so run `sb` again before you run git on the host. Snapshots are in
  `home/.sb/`.
- **Docker Desktop file sharing.** On macOS, Docker Desktop's VM can see every
  directory under Settings > Resources > File sharing, which defaults to all
  of `/Users`. Code that escapes a container lands in that VM. When it creates
  a container, `sb` warns if that list covers your home directory. Limit it to
  your project directories and this repo; nothing else is needed.

What this does **not** protect:

- The project directory is mounted read-write. The agent can wreck anything in
  it, so keep it committed and pushed.
- The agent's own credentials in `home/<agent>/` are mounted read-write,
  because the agent needs them. Anything it can reach on the public internet,
  it can reach with those.
- A container is still a shared kernel. This raises the cost of an escape a
  long way; it does not make it impossible.

`sudo apt-get install` no longer works inside the sandbox. Add the package to
`containers/base/Dockerfile` and run `sb <agent> --rebuild` instead.

## Layout

```
bin/sb                       the launcher
containers/base/             shared base image (sandbox-base) + sandbox-init
containers/<agent>/          Dockerfile (FROM sandbox-base) + sandbox.conf
containers/docker-compose.yml  older compose-based setup (mounts ../../src)
home/<agent>/                persisted agent config & credentials (gitignored)
```

### The base image

`containers/base/Dockerfile` (Ubuntu 26.04) provides everything the agents share:
- the unprivileged user `agent` (no sudo)
- curl, vim, emacs-nox, tmux, fish and git
- python3 with pip and venv (`python` → `python3`), plus node and npm
- iptables and iproute2, used by `sandbox-init`
- `~/.local/bin` on `PATH`

Agent Dockerfiles only add the agent itself.

### Agents

| agent      | `sandbox.conf` | runnable with `bin/sb` |
|------------|----------------|-----------------------------|
| `claude`   | yes            | yes                         |
| `gemini`   | yes            | yes                         |
| `omp`      | yes            | yes                         |
| `opencode` | yes            | yes                         |
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
   # USER agent
   ```

2. Create `containers/<name>/sandbox.conf`. It is bash, sourced by `bin/sb`:

   ```bash
   # host paths are relative to the repo root; created automatically
   CONFIG_MOUNTS=("home/<name>/.<name>:/home/agent/.<name>")
   AGENT_ENV=(SOME_VAR=value)
   AGENT_CMD=(<name> --some-flag)   # required
   ```

3. Run `sb <name> --rebuild`.

## Known issues

- The container user `agent` is hard-coded in `bin/sb` (`CUSER`) and in
  `containers/base/Dockerfile`.
- Containers created before the hardening keep their old, permissive settings.
  `sb` refuses to attach to them; fix it with `--recreate`.
- `ping` and other raw sockets do not work, because `NET_RAW` is dropped.
- The firewall blocks whole network ranges, not domains, so the agent can still
  talk to any public host. See [Isolation](#isolation) for what that means.
