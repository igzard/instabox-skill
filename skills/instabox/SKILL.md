---
name: instabox
description: Use when the user wants a cloud Linux machine (instabox.sh) — create, list, SSH into, exec in, pause/resume/stop/delete a machine, publish a port as a public HTTPS URL, save or start from a snapshot, manage Vault secrets or API keys, or drive instabox machines from code with the @insforge/instabox-sdk TypeScript SDK or the HTTP API.
---

# instabox

instabox gives you private Linux VMs ("machines", also called sandboxes) in the cloud. Each one has its own kernel, root and Docker. User `dev` with sudo; HOME is `/data/home`. Everything under `/data` and the installed software survive stops.

Docs: https://instabox.sh/docs · CLI help: `instabox --help`, `instabox <cmd> --help` (the installed CLI is the source of truth if this file disagrees).

## Setup

```sh
curl -fsSL https://instabox.sh/install.sh | sh   # install the CLI
instabox login                                    # browser sign-in (GitHub or Google)
instabox login --device                           # no browser on this computer
instabox whoami                                   # check who is signed in
```

`npx instabox@latest sh` does it all in one go: sign-in, a new machine, the CLI, and a shell.

Non-interactive use (CI, agents): `export INSTABOX_API_KEY=ibk_...` and the CLI needs no login. Keys are created in the dashboard (**API Keys**) or with `instabox keys create`. Never print, log or commit a key.

## Templates

`instabox templates` lists them. Default is `agent-browser`.

| Template | Use for |
|---|---|
| `agent-browser` | Linux + desktop (VNC) + browser — GUI / browser automation |
| `universal` | general dev box: git, gh, Node, Python, Go, Rust, Docker, Claude Code, Codex |
| `herdr` | the herdr client |
| `orca` | Orca desktop / mobile (`instabox orca connect <m>`) |

For ordinary coding work pass `--template universal` explicitly.

## Core commands

`<m>` is a machine name or ID (`m_...`).

```sh
instabox create --name dev --template universal   # waits until running
instabox ls                                       # machines and states
instabox get dev
instabox ssh dev                                  # interactive shell
instabox ssh dev -- 'cd /data/home/app && npm ci' # long command, no time limit
instabox exec dev -- uname -a                     # one command, returns its exit code
instabox exec dev --cwd /data/home --env K=V -- ls -la
echo data | instabox exec dev --stdin -- cat     # stdin, max 1 MiB
instabox files dev /data/home                     # list a dir or print a text file (live, as root)
instabox suspend dev && instabox resume dev       # pause: memory + processes kept
instabox stop dev && instabox start dev           # stop: processes end, disks kept
instabox rm dev                                   # delete machine and disks
instabox usage                                    # counts, storage, limits
```

`create` flags: `--template T | --snapshot S`, `--timeout 30m [--on-timeout pause|kill]`, `--auto-suspend 10m|0`, `--meta K=V` (repeatable), `--no-vault`, `--no-wait`. There are no size flags: every machine is 4 vCPU / 8 GiB.

### exec vs ssh

- `exec` runs `bash -lc` as `dev`, prints stdout+stderr when the command **ends** (no streaming), exits with the command's code. Timeout: `--timeout` seconds, default 60, max 600. Max 4 concurrent execs per account (more → retryable `429`).
- Anything longer (installs, builds, test suites, servers): use `instabox ssh <m> -- '<cmd>'`, or start it in the background / tmux.
- Read `~/MACHINE.md` and `~/AGENTS.md` in a fresh machine before working in it: they describe the box and its tools.

## Lifecycle

| State | Files | Memory & processes |
|---|---|---|
| running | kept | running |
| paused (`suspended`) | kept | kept |
| stopped | kept | lost — next start boots fresh |
| deleted | gone | gone |

- New machines **auto-pause after 30 idle minutes**. `exec`, `ssh` and `files` resume them automatically; a public port URL does **not**.
- `instabox auto-suspend <m> 2h` changes the idle time; `0` keeps it always on (needed for servers that must keep answering).
- `instabox timeout <m> 1h [--on-timeout pause|kill]` — one-shot deadline, 1 min to 24 h, `0` clears. A resumed machine has no timeout until you set one again.
- A paused machine cannot be stopped: resume or delete it.
- One operation at a time per machine; a second one gets `409 operation_conflict`.

## Public ports

```sh
instabox port publish dev 3000 --name 'Web app'   # prints https://3000-<id>.<domain>/
instabox port ls [dev]
instabox port unpublish dev 3000
```

Inside the machine: `instabox-port publish 3000` or `instabox-port serve ./dir --port 8080` (`instabox port setup <m>` installs it). The URL is **public, not secret** — anyone can reach the service until unpublished. It only answers while the machine runs, so set `auto-suspend 0` for anything that must stay up. Max 16 ports per machine.

## SSH details

```sh
instabox ssh-config dev        # ~/.ssh/config Host block (for IDEs, scp, rsync)
instabox ssh-token dev --ttl 1d  # a plain `ssh ibssh_...@host` line, no CLI/key needed
instabox ssh-token --revoke dev
instabox mosh dev              # needs mosh installed locally
```

Supported: shell, exec, pty, local port forwarding (`-L`). Not supported: agent forwarding, X11, remote forwarding. Connections close when the machine pauses/stops.

## Snapshots

A snapshot holds installed software and `/data` (home, workspace, saved gh/Claude/Codex/git/SSH sign-ins). No memory or processes.

```sh
instabox snapshot save dev --name dev-base [--root-only] [--no-wait]
instabox create --name dev2 --snapshot dev-base
instabox fork dev dev3                      # copy a stopped machine directly
instabox snapshot ls
instabox snapshot files dev-base /etc      # browse without starting a machine
instabox snapshot rm dev-base
```

`--root-only` leaves out `/data`. Saving may stop the machine first; deleting a machine keeps its snapshots.

## Vault (secrets)

Vault items are written into every machine at boot, start and resume: `env` items as env vars of login shells (so `exec` sees them), `file` items as files under HOME.

```sh
instabox vault set ANTHROPIC_API_KEY             # value from stdin or hidden prompt
instabox vault set-file .config/svc.json --from ./svc.json
instabox vault import .env
instabox vault set DATABASE_URL --selector project=shop   # only machines with --meta project=shop
instabox vault ls                                # names only, never values
instabox vault show dev                          # what this machine gets
instabox vault push dev                          # write items into a running machine now
```

Rules:
- Check for a credential **by name** (`vault show <m>`, or inside the machine `test -n "${NAME+x}"` / `test -f ~/path`) — never print values or cat secret files into output.
- A machine for another person / untrusted code: `create --no-vault` (or `start|resume <m> --no-vault`, permanent). Anything running in a Vault machine can read the secrets.
- Managing Vault items requires a sign-in; API keys cannot.

## API keys

```sh
instabox keys create --name ci --scope machines:read --scope machines:write --expires-in-days 30
instabox keys ls
instabox keys rm <key-id>
```

Scopes: `machines:read|write|delete`, `images:read|write|delete` (images = snapshots). Write/delete include read. Give `images:read` to keys that create machines (needed to resolve names). The secret is shown once.

## Driving machines from code

- TypeScript SDK (`@insforge/instabox-sdk`): see [references/sdk.md](references/sdk.md).
- Raw HTTP API (`https://api.instabox.sh/v1`): see [references/http-api.md](references/http-api.md).

## Limits (defaults)

5 machines in any state (paused/stopped count) · 10 snapshots · 200 GB storage · 20 API keys · 4 concurrent `exec`, 10 min each · timeout 1 min–24 h · 32 labels and 16 public ports per machine.

A `quota_exceeded` error means delete or stop something — check `instabox usage`, don't retry the same create in a loop.

## Habits

- Pick a name not already in `instabox ls` when the user asks for a new machine; reuse an existing one only when they want to continue earlier work.
- Give throwaway machines a deadline: `--timeout 1h --on-timeout kill`.
- Use `--meta` labels to group machines (`instabox ls --meta task=review`). Labels are plain text, not for secrets.
- Delete machines you created for a one-off task once done, if the user agrees.
- Problem with instabox itself (not the user's code)? `instabox feedback --type bug|feature-request|friction|other --component cli|api|sandbox|dashboard|skills|docs|other --title ... --detail ...`, then continue with a workaround.
