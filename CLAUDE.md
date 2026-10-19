# CLAUDE.md

Notes for coding agents working on this repo. Read README.md for what the
tool does and how to use it.

## What's here

- `bin/sb` is a single bash script that runs an agent in a per-project
  docker container attached to tmux.
- `containers/base/Dockerfile` builds the `sandbox-base` image. Every agent image is
  `FROM sandbox-base`, and `bin/sb` builds base automatically.
- `containers/<agent>/` contains a `Dockerfile` and a `sandbox.conf`. Only directories that
  have a `sandbox.conf` are runnable agents.
- `home/<agent>/` holds the agents' persisted config and **credentials**. It is
  gitignored. Never commit it, and never read or print its contents.

## Adding or changing an agent

- Dockerfile: start with `FROM sandbox-base` and add only the agent install. Base
  already has the user `satran`, sudo, tmux, git, python3, node/npm and
  `~/.local/bin` on PATH. Put shared tooling in `containers/base`, not in an agent
  image. If you install as root, switch back with `USER satran`.
- `sandbox.conf` is sourced as bash and may set:
  - `AGENT_CMD`, an array, **required**: the command run in tmux window 1
  - `CONFIG_MOUNTS`, an array of `"host:container"`; host is relative to the repo
    root, usually `home/<agent>/...`
  - `AGENT_ENV`, an array of `KEY=VALUE`
- The agent binary must be on the container PATH set in `bin/sb` (`CPATH`).
- Reference implementation: `containers/claude/`.

## Editing bin/sb

- Keep `set -euo pipefail`. Expand arrays that may be empty with
  `${A[@]+"${A[@]}"}` so `set -u` doesn't fail on old bash.
- Indent with tabs. Keep comments short, in the existing style.
- Run `bash -n bin/sb` and `shellcheck bin/sb` if they are available.
- If you change the CLI (options, usage text), update the Usage section of
  README.md to match. If you change how containers are named, mounted or built,
  update "How it works".
- Don't weaken the guard that refuses to mount `$HOME` or `/`.

## Testing

Docker may not be available inside a sandbox. When it is:

```sh
docker rmi sandbox-base sandbox-claude   # optional: force a cold build
bin/sb claude /tmp/sbx-test              # builds base, then claude; attaches
bin/sb list
bin/sb claude /tmp/sbx-test --stop
```

## Known issues

See the "Known issues" section of README.md. Fix them only when asked.
