# claude-container

Run [Claude Code](https://docs.claude.com/en/docs/claude-code) in an isolated macOS container using [Apple's `container`](https://github.com/apple/container) tool.

Call `claude-container` from any folder — it spins up a fresh Linux VM, copies your project in, launches Claude Code interactively, and dumps the resulting workspace back out into a timestamped folder when you quit. Your host filesystem stays untouched.

## Why

Claude Code is powerful, but giving an autonomous agent direct write access to your home directory is a lot of trust. Running it inside Apple's native container isolates the blast radius:

- No access to files outside the folder you launched from
- No access to your shell history, SSH keys, or other projects
- `--dangerously-skip-permissions` becomes dramatically less dangerous
- Changes only land on your real disk when you explicitly copy them from the output folder

## Requirements

- macOS with [Apple's `container`](https://github.com/apple/container) installed (`brew install --cask container` or via the installer)
- An `ANTHROPIC_API_KEY` in your environment (or log in interactively inside the container)

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/<you>/claude-container/main/claude-container \
  -o /usr/local/bin/claude-container
chmod +x /usr/local/bin/claude-container
```

Or clone and symlink:

```bash
git clone https://github.com/<you>/claude-container.git
ln -s "$PWD/claude-container/claude-container" /usr/local/bin/claude-container
```

## First-time setup (recommended)

Build a base image with Claude Code pre-installed so every session starts in seconds instead of waiting on `npm install`:

```bash
claude-container --build
```

This creates a `claude-code-base:latest` image (`node:20-slim` + `@anthropic-ai/claude-code`). You only need to do this once — re-run it whenever you want to refresh the Claude Code version.

## Usage

From any project folder:

```bash
cd ~/some-project
claude-container
```

Pass flags through to `claude` after `--`:

```bash
claude-container -- --dangerously-skip-permissions
claude-container -- --model claude-opus-4-6
```

## What happens

1. **Start** — a detached container named `claude-<folder>-<timestamp>` is launched (`sleep infinity` as PID 1, `--rm` on exit)
2. **Copy in** — current directory is tarred into `/workspace`, excluding `.git`, `node_modules`, `.venv`, `__pycache__`
3. **Run** — `container exec -it` drops you into Claude Code in `/workspace`
4. **Copy out** — when you quit Claude, `/workspace` is tarred back out to `./claude-session-<timestamp>/` in your original folder
5. **Cleanup** — container is stopped and removed

The original folder is never modified. To accept changes, diff against `./claude-session-<timestamp>/` and copy over what you want:

```bash
diff -r . claude-session-20260414-163000/
cp -r claude-session-20260414-163000/src/. src/
```

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | — | Forwarded into the container |
| `CLAUDE_CONTAINER_IMAGE` | `claude-code-base:latest` | Override the base image |

## Troubleshooting

**`container: command not found`** — install Apple's container tool first.

**Base image not found** — run `claude-container --build` once, or let it fall back to `node:20` (slower first run).

**Builder not running** — `container builder start` (the `--build` path does this automatically).

**Network issues inside container** — check `container system status` and Apple's docs; `container` uses a VM per container, so DNS/network config lives there.

## Caveats

- **Excluded paths** — `.git`, `node_modules`, `.venv`, `__pycache__` are not copied in (to keep startup fast). Git history in particular is absent; Claude can't run `git log` on your real history.
- **No host network mounts** — this is by design. If you need a DB socket or similar, extend the script with `-v` / `--mount` flags on `container run`.
- **Secrets** — only `ANTHROPIC_API_KEY` is forwarded. Add more `-e VAR` lines if your project needs them.
- **Arch** — defaults to `arm64` (Apple Silicon). Override with `container` flags if needed.

## License

MIT
