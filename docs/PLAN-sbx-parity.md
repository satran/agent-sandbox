# Plan: sbx-style features for `sb`

Status: proposal, not started. Work through it one phase at a time; each phase
is meant to be one branch / PR that leaves `sb` fully working.

## Goal

Keep `sb` (one readable bash script plus Dockerfiles) and add the parts of
Docker Sandboxes (`sbx`) that make it easier and safer to use:

| sbx feature | plan here | phase |
|---|---|---|
| Reach a sandbox as `<name>.sbx` over SSH, use it from Claude Desktop / VS Code | `<name>.sb` over SSH tunnelled through `docker exec`, no open port | 1 |
| Deny-by-default network policy by **domain**, logged | per-sandbox egress proxy on an internal network, policy files, `sb policy`, `sb blocked` | 2 |
| Credentials injected by a host-side proxy, never inside the sandbox | proxy injects tokens for selected hosts (GitHub, API keys) | 3 |
| Branch/mountless workspaces | `--workspace clone` and `--workspace none`, `sb pull` to bring work back | 4 |
| Agent state can't leak between sandboxes | read-only agent config, per-sandbox state | 5 |
| A separate VM per sandbox, Docker inside the sandbox | dedicated sandbox VM (easy), per-sandbox VM backend (spike) | 6 |
| `ls`/`rm`/`exec`/ports, kits, idle stop | subcommands, `sb forward`, `sb doctor`, kits | 0 and 7 |

### What we skip, and why

- **Credential injection for Claude subscription (OAuth) logins.** Claude Code
  manages and refreshes those tokens itself, and the request and refresh flow
  isn't a documented interface we can safely rewrite in a proxy. The token
  stays in `home/claude/` as it does today. Injection is done for API keys and
  GitHub tokens only (phase 3).
- **A microVM per sandbox on Docker Desktop.** Docker Desktop runs all
  containers in one shared VM, and we can't change that. Phase 6 offers a
  dedicated VM for all sandboxes (simple) and a spike on Apple's `container`
  runtime, which runs each container in its own VM.
- **Docker engine inside the sandbox** on the default backend. It needs
  `--privileged` or similar, which would undo the hardening. Only revisit it
  with a per-sandbox VM (phase 6b).
- **Writing connections straight into Claude Desktop's config** (sbx's
  `--auto-create`). The app's config format isn't documented. `sb` prints the
  host name to type into "+ Add SSH connection" instead.
- **Non-HTTP egress** (SSH to git hosts, raw TCP/UDP, ping). It stays blocked
  by design, like sbx. Use git over HTTPS (phase 3 makes that work without a
  token in the sandbox).

## Ground rules for the implementer

- Follow `CLAUDE.md`. This plan is the explicit request for the new mounts,
  containers and sshd it describes; don't add anything beyond it without
  asking.
- Never weaken the existing guarantees: no sudo, `--cap-drop ALL`,
  `no-new-privileges`, no docker socket, no SSH agent, the `$HOME`/`/` mount
  guard, the tmpfs over `home/`, the filtered gitconfig, the guard on files the
  host executes.
- Everything sb keeps on the host lives under `home/.sb/` (gitignored, never
  mounted into a sandbox unless a phase says so, and then read-only and
  narrowly).
- Keep the existing CLI working: `sb <agent> [path] [options] [-- args]` and
  `sb list` must behave as today. New commands are subcommands; refuse agent
  directory names that collide with a subcommand.
- `bash -n bin/sb` and `shellcheck bin/sb` after every change. Update README
  Usage / How it works / Isolation in the same PR as the code.
- Docker usually isn't available inside a sandbox. When developing inside one,
  run the static checks and ask the user to run `tests/run.sh` on the host.

## Phase 0 — foundations (small)

Goal: names, subcommands and a test harness the later phases build on.

1. **Sandbox names.** Add `--name NAME` (`[a-z0-9][a-z0-9-]*`, max 40).
   Default: `<agent>-<basename of project>`, plus `-<hash>` only on collision.
   Store it in a `sandbox.name` label. The container name stays as today, so
   existing containers keep working; an existing container without the label
   gets the default name.
2. **Subcommands**, all taking a sandbox name (or agent + path as today):
   - `sb list` adds NAME, SSH HOST and WORKSPACE columns.
   - `sb shell NAME` opens a bash in the project (`docker exec -it`), without
     tmux.
   - `sb stop NAME`, `sb rm NAME` (with confirmation unless `-f`; also removes
     `home/.sb/` state for that sandbox and, from phase 2, its proxy and
     network). Keep `--stop` as an alias.
   - `sb help`.
3. **Resolve `docker` by absolute path** (`command -v docker`, then
   `/usr/local/bin`, `/opt/homebrew/bin`, `~/.docker/bin`). Apps started from
   the Dock (Claude Desktop) have a minimal PATH, and phase 1 runs `sb` from
   them.
4. **Tests.** Add `tests/run.sh` (host-side, needs docker): creates a sandbox
   for a temp project, runs the checks from `CLAUDE.md` Testing through
   `docker exec`, and prints pass/fail. Every later phase adds its checks here.

Done when: existing usage is unchanged, `sb list` shows names, `tests/run.sh`
passes.

## Phase 1 — SSH access and Claude Desktop

Goal: `ssh claude-myproject.sb` works, and Claude Desktop can use it as an SSH
environment. Nothing listens on a network port; the transport is
`docker exec`, exactly like sbx's `ProxyCommand`.

### In the image

- `containers/base/Dockerfile`: install `openssh-server` (the setuid sweep
  already runs after installs; keep it last).
- Add `containers/base/sb_sshd_config`, root-owned, readable by `agent`:

  ```
  # sshd runs as the 'agent' user in inetd mode (-i), started by sb ssh-proxy.
  HostKey /home/agent/.local/state/sb-ssh/host_ed25519
  AuthorizedKeysFile /etc/sb/authorized_keys
  StrictModes no
  PubkeyAuthentication yes
  PasswordAuthentication no
  KbdInteractiveAuthentication no
  UsePAM no
  PermitRootLogin no
  AllowUsers agent
  AllowAgentForwarding no
  AllowTcpForwarding local      # -L for dev-server previews; never -R
  PermitListen none
  AllowStreamLocalForwarding no
  GatewayPorts no
  X11Forwarding no
  PermitTunnel no
  PermitUserEnvironment no
  Subsystem sftp internal-sftp
  PidFile none
  LogLevel ERROR
  ```

  Verify that non-root `sshd -i` works with the Ubuntu 26.04 OpenSSH
  (privilege separation, `sshd-session`). If it doesn't, stop and report:
  don't run sshd as root.

### On the host

- `sb ssh setup` (run once, safe to re-run):
  - creates the client key `home/.sb/ssh/id_ed25519` (0600). Only the `.pub`
    is ever mounted, at `/etc/sb/authorized_keys:ro`;
  - writes `home/.sb/ssh/config` and adds one `Include` line at the very top
    of `~/.ssh/config` (back up the file first; idempotent; marked
    `# >>> sb (managed) >>>`):

    ```
    Host *.sb
        User agent
        ProxyCommand "<abs>/bin/sb" ssh-proxy %h
        IdentityFile "<repo>/home/.sb/ssh/id_ed25519"
        IdentitiesOnly yes
        IdentityAgent none
        ForwardAgent no
        ForwardX11 no
        ControlMaster no
        ControlPath none
        UserKnownHostsFile "<repo>/home/.sb/ssh/known_hosts"
        KnownHostsCommand "<abs>/bin/sb" ssh-known-hosts %H
        StrictHostKeyChecking yes
    ```
  - `sb ssh remove` undoes both.
- `sb ssh-proxy HOST` (internal; called by ssh):
  - maps `NAME.sb` to the container through the `sandbox.name` label. If it is
    stopped, start it and `wait_ready`; if it doesn't exist, fail with
    "create it with `sb <agent> <path> --name NAME`";
  - runs `guard_check`, then `docker exec -i --user agent CTR` a small
    wrapper that creates the host key on first use (`ssh-keygen -q -t
    ed25519 -N ''`) and execs `/usr/sbin/sshd -i -f /etc/ssh/sb_sshd_config`
    with `-o SetEnv=...` for `PATH`, `HOME`, `USER` and the agent's
    `AGENT_ENV`. sshd starts sessions with a clean environment, so without
    this, `CLAUDE_CONFIG_DIR` and `PATH` are lost;
  - runs `guard_check` again when the session ends. Since an app's stderr is
    not visible, a warning also raises a macOS notification (`osascript`),
    when available.
- `sb ssh-known-hosts HOST` prints `HOST <pubkey>` read from the container.
  Authenticity comes from `docker exec`, not from the network, so this is safe.
- Existing containers get the new mounts only on `--recreate`. Say so when
  `ssh-proxy` finds the mount missing.

### Usability

- After `sb claude ~/src/app`, print once: "Claude Desktop: + Add SSH
  connection → host `claude-app.sb`, folder `/home/agent/src/app`."
- `sb forward NAME PORT[:HOSTPORT]` runs `ssh -N -L` for dev servers.
- README section "Use with Claude Desktop / VS Code".

### Checks (add to tests/run.sh)

- `ssh NAME.sb true` succeeds; `ssh -R 9999:localhost:22 NAME.sb true` is
  refused; `ssh -A NAME.sb 'test -z "$SSH_AUTH_SOCK"'`;
  `ssh NAME.sb 'echo $CLAUDE_CONFIG_DIR'` prints the configured value; `id`
  over ssh shows no groups; a stopped sandbox is started by `ssh`.
- Manual: Claude Desktop connects, opens the project folder, runs a command.
  Note what Desktop installs on the remote, and which hosts it needs. Phase 2
  must allow them.

## Phase 2 — domain-based egress proxy

Goal: the sandbox can reach only the domains its policy allows. Everything
else, including DNS to the outside, is impossible. Every request is logged.
This closes the "any public host" known issue.

### Network layout, per sandbox

```
 sandbox container ──(internal network sb-net-<hash>, no route out)──┐
                                                                      │
                               proxy container  sb-proxy-<hash>  ◄────┘
                                     │  (also on 'sandbox-egress', a normal
                                     ▼   bridge with icc disabled)
                                  internet
```

- `docker network create --internal sb-net-<hash>` per sandbox. The sandbox
  joins only this network, so there is no default route and no external DNS.
  Remove `--dns` for proxied sandboxes. The shared `sandbox` network is kept
  only for `SB_PROXY=off`.
- The proxy container joins `sb-net-<hash>` (alias `proxy`) and
  `sandbox-egress`.
- Environment in the sandbox: `HTTP_PROXY`, `HTTPS_PROXY` (and lowercase)
  `=http://proxy:3128`, `NO_PROXY=localhost,127.0.0.1,::1`.
- `sandbox-init` (defence in depth): with a proxy, OUTPUT allows only lo,
  established, and TCP to the proxy's address on 3128; everything else is
  rejected. Without a proxy, today's rules apply.
- `sb` starts, stops and removes the proxy together with its sandbox
  (`ssh-proxy` and attach start it too).

### Proxy implementation

- New image `containers/proxy/` (not FROM sandbox-base; small Debian/Ubuntu
  with a pinned `mitmproxy` in a venv). Runs `mitmdump` with an addon
  `sb_policy.py`:
  - On `http_connect` and `request`: match the host against the allowlist
    (exact `host`, or `*.domain`; ports 80/443 unless the policy says
    otherwise). Deny with a 403 whose body says
    `blocked by sb policy: <host>. On the host run: sb policy allow <name> <host>`,
    which tells both the agent and you what to do.
  - Resolve the name inside the proxy and refuse private, loopback,
    link-local, CGNAT and metadata addresses after resolution (DNS rebinding),
    then connect to that checked address.
  - TLS is **passed through, not intercepted**, for every host without an
    injection rule (phase 3), via `tls_clienthello` → ignore connection. The
    proxy sees only the host name (CONNECT / SNI).
  - Log one JSON line per decision (time, host, port, method, allow/deny,
    rule) to `/var/log/sb/proxy.jsonl`, mounted from
    `home/.sb/logs/<name>/`.
  - Reload the policy when the file changes (mtime check) or on SIGHUP.
- Hardening: proxy-init runs as root only to install iptables that reject the
  private ranges (same list as `sandbox-init`), then drops to user `proxy`
  with `setpriv --reuid --regid --clear-groups --inh-caps=-all` and execs
  mitmdump. Run with `--cap-drop ALL --cap-add NET_ADMIN --cap-add SETUID
  --cap-add SETGID` (dropped at exec), `--read-only`, `--tmpfs /tmp`,
  `no-new-privileges`, a pids limit. No project mount, ever.

### Policies

- Built-in, versioned in the repo under `policies/`:
  - `deny-all` (only what the agent itself needs, see below);
  - `balanced` (default): package registries and code hosts, for example
    `github.com`, `*.githubusercontent.com`, `codeload.github.com`,
    `registry.npmjs.org`, `pypi.org`, `files.pythonhosted.org`,
    `proxy.golang.org`, `crates.io`, `static.crates.io`, `index.crates.io`.
    Review the list in the PR;
  - `allow-all` (logs everything, still blocks private addresses).
- Each agent declares what it needs to function, in `sandbox.conf`:
  `ALLOW_HOSTS=(api.anthropic.com claude.ai ...)`. Find the exact list by
  running each agent under `deny-all` and reading `sb blocked`. These are
  always added.
- Per-sandbox overrides live on the host only:
  `home/.sb/policy/<name>.conf` (`base balanced` + `allow host` / `deny host`
  lines). Never read policy from the project directory, because the agent can
  write it.
- Global default in `home/.sb/config` (`SB_POLICY=balanced`); `--policy NAME`
  when creating a sandbox.

### CLI

- `sb policy show NAME`, `sb policy set NAME balanced|deny-all|allow-all`,
  `sb policy allow NAME HOST...`, `sb policy deny NAME HOST...`. These apply
  live, with no restart.
- `sb blocked NAME [-f]` shows recent denied hosts, newest first, with counts.
  `sb blocked NAME --allow` asks y/n for each and adds the answers. This is
  the "it didn't work, why?" path, and it has to be quick.
- `sb log NAME [-f]` shows all proxy decisions.
- `SB_PROXY=off` for debugging, with a loud warning, like `SB_FIREWALL=off`.

### Checks

From inside the sandbox, under `balanced`:
- `curl -sI https://github.com` works; `curl -sI https://example.com` gets
  403 with the sb message;
- `getent hosts example.com` fails (no DNS out);
  `curl --noproxy '*' -m 5 https://github.com` fails;
  `curl -m 5 http://1.1.1.1` fails; `nc -w 3 1.1.1.1 53` fails;
- an allowed name that resolves to a private address is refused (add
  `localtest.me`, which resolves to 127.0.0.1, to a test policy);
- `pip download requests`, `npm view react version` and `git clone https://github.com/...`
  work through the proxy;
- `claude` starts and answers; `sb blocked` shows the denials.
Plus all existing checks from `CLAUDE.md`.

## Phase 3 — secrets and credential injection

Goal: the agent can push to GitHub and call APIs without the token ever
entering the sandbox.

- `sb secret set NAME` reads a value from stdin and stores it in the macOS
  Keychain (`security add-generic-password -s sb -a NAME -U -w`). On Linux,
  use `home/.sb/secrets/NAME` (0600). Also `sb secret ls` and
  `sb secret rm NAME`. Values are never printed.
- Injection rules in the sandbox policy:

  ```
  inject github.com      header Authorization "Basic {b64:x-access-token:{secret:github}}"  path ^/myorg/
  inject api.github.com  header Authorization "Bearer {secret:github}"
  inject api.anthropic.com header x-api-key "{secret:anthropic}"   # API-key users only
  ```

  The optional `path` regex (and `method`) limits where a token may be used.
  For example, push only to repos under one org. Injection stops theft, not
  misuse, so recommend fine-grained, repo-scoped tokens in the README.
- At sandbox start, `sb` reads the needed secrets and writes them to
  `home/.sb/run/<name>/secrets.json` (0600). That file is mounted read-only
  into the **proxy only**, and deleted when the sandbox stops. Don't put
  secrets in container environment variables (`docker inspect` shows them).
- TLS interception happens **only** for hosts with an inject rule:
  - `sb` creates a CA once in `home/.sb/proxy/ca/` (key 0600, never mounted
    into a sandbox; the proxy gets it read-only);
  - the sandbox gets only the CA certificate, read-only.
    `sandbox-init` (root, at start) runs `update-ca-certificates`, and `sb`
    sets `NODE_EXTRA_CA_CERTS`, `SSL_CERT_FILE` and `REQUESTS_CA_BUNDLE`.
- Placeholders inside the sandbox so tools send auth at all: `GH_TOKEN=sb-proxy`,
  and a git credential helper in the base image that returns `x-access-token` /
  `sb-proxy` for hosts that have inject rules. The proxy overwrites the
  header.
- Checks: `git push` to an allowed repo works; `env` and the filesystem
  contain no real token (`grep -r` for the token value across `/home/agent`
  and `/proc/*/environ` finds nothing); a push outside the `path` rule is
  refused with a clear message; hosts without rules are still passed through
  (the certificate issuer is the real CA, not sb's).

## Phase 4 — workspace modes

Goal: let the agent work on a copy, so it can never write to the host
project. That makes the host-executed-file guard unnecessary for that
sandbox.

- `--workspace mount` (default; today's behaviour).
- `--workspace clone`:
  - On the host, `git -C PROJECT bundle create - --all` (plus a WIP commit of
    uncommitted changes on a temp ref if `--include-dirty`) is streamed into
    the container and cloned into a named volume `sb-ws-<hash>` at the same
    mirrored path. No host `.git` config or hooks are copied, and nothing
    else from the host is visible. Ignored files such as `.env` are left out.
  - `sb pull NAME [--commit]`: optionally commits the sandbox's working tree
    on its current branch, then streams `git bundle create - <branches>` out
    of the container and runs `git fetch <bundle> 'refs/heads/*:refs/remotes/sb/<name>/*'`
    on the host. Fetching from a bundle runs no hooks. You review and merge
    with normal git (`git diff main sb/<name>/main`).
  - `sb push NAME` sends newer host commits into the sandbox the same way.
- `--workspace none`: an empty volume at `/home/agent/workspace`. The agent
  clones what it needs through the proxy. Same as sbx's mountless sandboxes.
- Workspace mode is fixed at creation and changed only with `--recreate`,
  like mounts. `sb rm` asks before deleting a volume with unpulled commits.
- The guard runs only for `mount`; `sb list` shows the mode.
- README: recommend `clone` for unattended or long runs, `mount` for
  interactive work.

## Phase 5 — keep agent config from leaking between sandboxes

Problem: `home/claude/.claude` is mounted read-write into **every** Claude
sandbox. An agent in one project can write `settings.json` hooks, plugins,
skills, commands or a global `CLAUDE.md`, and those then run in every other
sandbox.

Spike first, then implement:
- Mount config that should not change at run time read-only, on top of the
  read-write dir: `settings.json`, `CLAUDE.md`, `plugins/`, `skills/`,
  `agents/`, `commands/`, `hooks/` (only the ones that exist).
- Move the plugin install out of `claude-start` into a host-side
  `sb claude --setup` (or image build), so nothing needs to write plugins at
  run time.
- Keep the credential file and per-session state writable. Test whether
  Claude Code writes credentials by atomic rename, which breaks single-file
  bind mounts. If it does, keep the directory writable and only the listed
  paths read-only.
- `sb config edit claude` edits the read-only parts on the host.
- Check: from a sandbox, writing to `~/.claude/settings.json` fails; login,
  `/resume` and token refresh still work across a restart.

Apply the same pattern to the other agents' `CONFIG_MOUNTS` where it fits.

## Phase 6 — stronger VM isolation (optional)

### 6a. A dedicated VM for sandboxes (easy)

- `SB_DOCKER_CONTEXT` (and `sb doctor` advice). Run sandboxes in a separate
  Docker VM, for example a Colima profile that shares only `~/src` and this
  repo, so an escape lands in a VM holding only sandboxes and those folders.
  This replaces the Docker Desktop File sharing advice.
- Optional, opt-in only with a clear warning: `--docker`, a per-sandbox
  privileged `docker:dind` sidecar on the sandbox's internal network
  (`DOCKER_HOST=tcp://docker:2375`; its egress also through the proxy).
  Allowed **only** when the context is a dedicated sandbox VM. Refuse it on
  Docker Desktop.

### 6b. A VM per sandbox (spike)

- Apple's `container` runtime runs each container in its own lightweight VM
  on Apple's Virtualization framework (Apple silicon; full networking needs
  macOS 26; no Compose). Spike only:
  - Do our images, `sandbox-init`/iptables, internal networking, labels and
    `exec -i` (for SSH) work with it?
  - Which `docker run` flags have no equivalent?
- If the spike is positive: put every runtime call in `bin/sb` behind a small
  `rt_*` function layer, add `SB_RUNTIME=docker|container`, and then allow
  `--docker` with the per-sandbox VM. If it's negative, document why and stop
  at 6a.

## Phase 7 — polish

- `sb doctor`: checks docker, its path, the VM's file sharing, the SSH
  include, the proxy image, the policy and Keychain secrets, and prints fixes.
- Idle stop: `sb gc` stops sandboxes with no tmux client and no ssh session
  for `SB_IDLE` minutes. Optionally a launchd agent installed by
  `sb gc --install`.
- Kits: `containers/kits/<kit>.Dockerfile` fragments (`go`, `rust`, `java`,
  ...) and `--kit go,rust`. They build a derived image
  `sandbox-<agent>-<kits-hash>` and can extend the policy
  (`kits/<kit>.allow`).
- Shell completion for subcommands and sandbox names.

## Suggested order and size

| phase | value | effort | depends on |
|---|---|---|---|
| 0 foundations | enables the rest | S | — |
| 1 SSH / Desktop | big usability win | M | 0 |
| 2 egress proxy | biggest security win | L | 0 |
| 3 secrets | security + usability | M | 2 |
| 4 workspaces | security | M | 0 (1 for convenience) |
| 5 config isolation | security | S–M | — |
| 6a dedicated VM | security | S | — |
| 6b VM per sandbox | security | spike | 2 |
| 7 polish | usability | S each | — |

## Decisions for the user before starting

1. Default policy: `balanced` (recommended) or `deny-all`.
2. Default workspace mode: keep `mount`, or make `clone` the default.
3. Default sandbox name: `<agent>-<project>` OK?
4. Secrets: macOS Keychain (recommended) or files under `home/.sb/secrets/`.
5. Proxy per sandbox (recommended: isolated logs and policy, small overhead)
   or one shared proxy.
