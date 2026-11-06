# CLAUDE.md

Notes for coding agents working on this repo. Read README.md for what the
tool does and how to use it.

## What's here

- `bin/sb` is a single bash script that runs an agent in a per-project
  docker container, attached to the user's terminal with `docker exec -it`.
  Closing the window ends the agent; `sb` kills each run's processes by
  their `SB_RUN` tag. `sb shell` opens a bash there instead.
- `containers/base/Dockerfile` builds the `sandbox-base` image. Every agent image is
  `FROM sandbox-base`, and `bin/sb` builds base automatically.
- `containers/base/sandbox-init` is the container's main process (under
  docker's `--init`). It runs as root,
  installs the egress firewall, then idles. The agent is attached with
  `docker exec --user agent` and never has those privileges.
- `containers/<agent>/` contains a `Dockerfile` and a `sandbox.conf`. Only directories that
  have a `sandbox.conf` are runnable agents.
- `home/<agent>/` holds the agents' persisted config and **credentials**. It is
  gitignored. Never commit it, and never read or print its contents.
  `home/.sb/` is `sb`'s own host-side state (guard snapshots, the filtered
  gitconfig); it lives there so it is never visible to a sandbox.

## Adding or changing an agent

- Dockerfile: start with `FROM sandbox-base` and add only the agent install. Base
  already has the user `agent`, git, python3, node/npm, iptables and
  `~/.local/bin` on PATH. Put shared tooling in `containers/base`, not in an agent
  image. If you install as root, switch back with `USER agent`.
- There is no sudo in the image and the runtime drops all capabilities, so an
  agent cannot install system packages at run time. Anything it needs belongs in
  `containers/base/Dockerfile`. Do not add `sudo` back, and do not add
  capabilities or mounts to `bin/sb` without being asked.
- `sandbox.conf` is sourced as bash and may set:
  - `AGENT_CMD`, an array, **required**: the command `sb` runs
  - `CONFIG_MOUNTS`, an array of `"host:container"`; host is relative to the repo
    root, usually `home/<agent>/...`
  - `AGENT_ENV`, an array of `KEY=VALUE`
  - `AGENT_SINGLE_RUN=1` to refuse a second run in the same container
- `CONFIG_MOUNTS` must include where the agent stores its conversations, so
  its resume flag still works after `--recreate`. Add the flag to the
  "resume with" column of the Agents table in README.md.
- The agent binary must be on the container PATH set in `bin/sb` (`CPATH`).
- Reference implementation: `containers/claude/`.

## Editing bin/sb

- Keep `set -euo pipefail`. Expand arrays that may be empty with
  `${A[@]+"${A[@]}"}` so `set -u` doesn't fail on old bash.
- Indent with tabs. Keep comments short, in the existing style.
- Run `bash -n bin/sb` and `shellcheck bin/sb` if they are available.
- If you change the CLI (options, usage text), update the Usage section of
  README.md to match. If you change how containers are named, mounted or built,
  update "How it works". If you change the sandbox boundary, update "Isolation".
- Don't weaken the guard that refuses to mount `$HOME` or `/`, the tmpfs that
  hides `home/` when the project contains this repo, or the `--cap-drop` /
  `no-new-privileges` / firewall settings.
- Every `docker exec` must pass `--user "$CUSER"`: the container runs as root
  for the firewall. Only the listed terminal variables may be passed to it.

## Testing

Docker may not be available inside a sandbox. When it is:

```sh
docker rmi sandbox-base sandbox-claude   # optional: force a cold build
bin/sb claude /tmp/sbx-test              # builds base, then claude; runs it
bin/sb shell claude /tmp/sbx-test        # bash in the same container
bin/sb list
bin/sb claude /tmp/sbx-test --stop
```

`sb` needs a terminal. To test it without one, run it under a pty (e.g.
python's `pty.fork`) and close the pty to simulate closing the window; `ps` in
the container should then show nothing tagged `SB_RUN` left.

After changing anything security related, check from inside the sandbox that
`id` shows no groups, `sudo` is absent, `curl -m 5 http://192.168.1.1` and
`curl -m 5 http://169.254.169.254` fail, and `curl -sI https://github.com`
still works. `docker logs <container>` shows what `sandbox-init` did.

## Known issues

See the "Known issues" section of README.md. Fix them only when asked.
