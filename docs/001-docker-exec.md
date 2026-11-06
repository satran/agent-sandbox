# Attach agents with `docker exec` instead of tmux

*   **Author:** Satyajit Ranjeev
*   **Status:** Draft
*   **Date:** 2026-11-06

--------------------------------------------------------------------------------

## Overview

`sb` runs coding agent CLIs (Claude Code, Antigravity, omp, opencode) in Docker
containers, one container per agent and project, so they can run with
permission prompts turned off without touching the rest of the machine. Today
`sb` attaches the user to a tmux session inside the container. The agent runs
in tmux's first window and a spare shell runs in its second.

This design removes tmux. `sb <agent> [path]` will start the agent with
`docker exec -it --user agent`, so the agent's terminal is the user's iTerm tab
with nothing in between. Closing the tab ends that agent, and the user picks
the conversation up again with the agent's own resume flag (for example
`claude --continue`). A new `sb shell <agent> [path]` command opens a shell in
the same container. The container model (one per agent and project), the
firewall and the other security settings stay as they are.

## The Problem

tmux in the container exists for two reasons: the agent keeps running if the
terminal window is closed by accident, and there is always a second window
with a shell in the container. Both have a cost the user is no longer willing
to pay.

**tmux stands between the agent and the terminal.** tmux is a terminal
emulator of its own. It reads everything the agent prints, keeps its own copy
of the screen, and redraws that copy in iTerm. The agent sees tmux's terminal
(`TERM=tmux-256color`), not iTerm's. As a result, iTerm features that agent
CLIs rely on work only partly, or only after tmux settings are tuned for them:
- keyboard combinations such as Shift+Enter;
- scrolling back through and searching the agent's output with iTerm's own
  scrollback;
- selecting and copying text;
- the clipboard over escape sequences (OSC 52);
- desktop notifications;
- links and images.

`containers/tmux.conf` already turns on extended keys, passthrough, the
clipboard and true colour, and the agents still don't behave the way they do on
the host. The user's goal is the full Claude Code experience inside the
sandbox, and tmux is in the way of that.

**The persistence tmux offers is not needed.** The user does not use tmux
splits, extra windows, copy mode or several viewers at once. The only case
that mattered was closing a window by accident. For that case it is enough to
reopen the agent and continue the conversation, which the agents can already
do from their saved history. Restoring the old screen is not needed.

**Removing tmux is not free.** It brings three problems that tmux was hiding:
- **A closed window may leave the agent running.** With `docker exec`, the
  agent's terminal belongs to Docker inside the container, not to the window.
  An agent left running with no window, and with permission prompts off, keeps
  spending tokens and changing files. Today tmux at least limits this to one
  agent per container.
- **A plain `docker exec` shell is root.** The container is started with
  `--user 0:0` so its first process can install the firewall, and that root
  keeps `NET_ADMIN`. If the user replaces tmux's shell window by typing
  `docker exec` by hand, they get a shell that can turn the firewall off.
- **The container has nothing that cleans up finished processes.** Its first
  process ends in `exec sleep infinity`. Starting and stopping agents often
  leaves zombie processes, which count toward the 2048-process limit.

The design has to solve these, or the change trades better terminal behaviour
for a weaker sandbox.

## Goals

*   **The agent sees iTerm directly.** Inside the container, the agent behaves
    the same as it does on the host. Test this in iTerm for every agent:
    *   Shift+Enter, Ctrl-P and Ctrl-C reach the agent;
    *   true colour, emoji and wide characters render correctly;
    *   the mouse and focus events work;
    *   the window size is right at start and after a resize;
    *   `printf '\e]52;c;%s\a' "$(printf sbtest | base64)"` puts `sbtest` in
        the Mac clipboard;
    *   `printf '\e]9;sbtest\a'` shows an iTerm notification.
*   **Closing the window leaves nothing behind.** Close the window in the
    middle of a task. Within a few seconds, `ps` in the container shows no
    agent process, none of its child processes (D-Bus, keyring, MCP servers)
    and no zombie processes.
*   **Resume is one command away.** `sb <agent> [path] -- <resume flag>`
    reopens the previous conversation for every agent that has a resume flag,
    and the README lists the flag for each agent.
*   **A shell is one `sb` command away and is never root.**
    `sb shell <agent> [path]` opens a shell as `agent` in the project folder.
    Every command `sb` runs in a container passes `--user agent`, and there is
    no option to choose another user.
*   **Old containers fail clearly, not silently.** A container created before
    this change either keeps working or `sb` prints a message telling the user
    to run `--recreate`. It never starts with a mount that has become an empty
    directory, and never leaves an old tmux agent running unseen.
*   **The sandbox boundary does not change.** The security checks in CLAUDE.md
    pass in both the agent and `sb shell`:
    *   `id` shows no groups;
    *   `sudo` is absent;
    *   the private-range and metadata `curl` calls fail;
    *   `curl -sI https://github.com` works.

    In addition, only a fixed, documented list of environment variables enters
    the container from the host.
*   **`bin/sb` gets simpler.** The tmux block, the tmux config mount and the
    "session already running, ignoring agent args" branch are removed.

## Non-Goals

*   **Keeping the agent running after its window closes.** A closed window ends
    the agent. Resume replaces persistence.
*   **Restoring the previous screen when the user comes back.**
*   **Terminals other than iTerm.** Ghostty and kitty should work too, since
    nothing here is specific to iTerm except the badge, but they are not
    tested and not a goal.
*   **Splits, extra windows, or several viewers on one agent.**
*   **Features the container itself blocks.** These are not caused by tmux and
    go in the README's known issues:
    *   pasting an image from the Mac clipboard;
    *   dragging Mac files in (their paths differ in the container);
    *   the agent opening a browser on the host;
    *   AppleScript;
    *   Claude Code's `/terminal-setup`.
*   **Running the container's first process as non-root.** It stays root to
    install the firewall. Installing the firewall from a helper container
    instead changes the sandbox boundary and needs its own design.
*   **Two Antigravity sessions in one container.** A second run is refused
    while the shared keyring has not been shown to cope with two sessions.
*   **A `--resume` option in `sb`.** The resume flag is passed as an agent
    argument, as it can be today.

## Open Questions

*   **Does a `docker exec` process survive its client going away?** Most
    reports say yes: the agent keeps running when the window closes. This
    decides whether `sb` must kill the agent itself. It has to be checked on
    the user's Docker Desktop before the cleanup work starts.
*   **What `TERM` does `docker exec -t` give by default?** If it is `xterm`,
    `sb` must always pass `TERM` from the host.
*   **Does Antigravity (`agy`) have a resume flag?** If not, closing an
    Antigravity window loses the conversation. Check `agy --help` in the image.
*   **What is omp's resume flag?** It is probably `--continue` or `--resume`.
    Check `omp --help` in the image.
*   **Does the window size arrive at exec start, or only after the first
    resize?** If only after, the agent's first screen is drawn at the wrong
    size. This needs a test, and possibly a minimum Docker Desktop version.

<!-- Sections below are added in later steps. -->

*Drafted with EGM*
