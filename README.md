# claude-sync

> Sync your [Claude Code](https://docs.anthropic.com/en/docs/claude-code) config and project memory across devices via git.

**The problem.** Claude Code keeps all your settings, custom skills, MCP servers, and per-project memory locally under `~/.claude/`. If you use Claude Code on more than one machine — laptop and desktop, mac and Linux, work and home — you lose context every time you switch. Memory you wrote on one device isn't visible on the other.

**The fix.** A tiny bash tool that symlinks a git-backed config directory into `~/.claude/`. Push from one device, pull from the next, and your full Claude Code state moves with you.

## What gets synced

| Item | Synced |
|---|---|
| `settings.json` (permissions, plugins, status line) | ✓ |
| `keybindings.json` | ✓ |
| Global `CLAUDE.md` | ✓ |
| Custom skills (`skills/`) | ✓ |
| Custom MCP servers (`mcp-servers/`) | ✓ (code only; device-local `.venv/` is preserved) |
| Project memory (`projects/*/memory/`) | ✓ |
| Conversation logs, cache, sessions, telemetry | ✗ (stay local) |

## How it works

Claude Code derives project directory names in `~/.claude/projects/` from absolute paths:

```
~/.claude/projects/-Users-alice-dev-myapp/       # macOS
~/.claude/projects/-home-alice-dev-myapp/        # Linux
```

These differ across devices because `$HOME` differs. `claude-sync` solves this by:

1. Storing each project under a **canonical name** in your repo (home prefix stripped): `-myapp`.
2. Creating **device-specific symlinks** on each machine that point back to the canonical memory directory.

Settings, skills, `CLAUDE.md`, and MCP servers are symlinked from `~/.claude/` directly into your repo.

## Install

One-line installer:

```bash
curl -fsSL https://raw.githubusercontent.com/oliverzli/claude-sync/main/install.sh | bash
```

Or clone manually:

```bash
git clone https://github.com/oliverzli/claude-sync.git ~/.local/share/claude-sync
ln -sfn ~/.local/share/claude-sync/bin/claude-sync ~/.local/bin/claude-sync
# make sure ~/.local/bin is on your $PATH
```

Requires `bash`, `git`. Tested on macOS and Linux.

## Quick start

### First device

1. **Create a private git repo for your config.** GitHub, GitLab, self-hosted — anywhere git works. **Make it private** — it will contain your settings (possibly with tokens) and your project memory.

2. **Initialize it locally:**

   ```bash
   claude-sync init ~/dotfiles/claude-config
   ```

   This creates the directory, runs `git init`, writes a `.gitignore` allowlisting only what should sync, and drops a `.claude-sync-data` marker.

3. **Point `claude-sync` at it** (add to your shell profile):

   ```bash
   export CLAUDE_SYNC_DIR=~/dotfiles/claude-config
   ```

4. **Add your private remote:**

   ```bash
   git -C ~/dotfiles/claude-config remote add origin git@github.com:you/claude-config.git
   ```

5. **Symlink into `~/.claude/`:**

   ```bash
   claude-sync link
   ```

   If the device already has local Claude data, `link` migrates existing project memory into the repo automatically.

6. **Push:**

   ```bash
   claude-sync sync
   ```

### Additional devices

```bash
# install the tool (same one-liner as above)
curl -fsSL https://raw.githubusercontent.com/oliverzli/claude-sync/main/install.sh | bash

# clone your config repo
git clone git@github.com:you/claude-config.git ~/dotfiles/claude-config
export CLAUDE_SYNC_DIR=~/dotfiles/claude-config   # add to .zshrc / .bashrc

# link + pull
claude-sync link
claude-sync pull
```

## Commands

```
claude-sync init <dir>   create a new data repo at <dir>
claude-sync              sync (pull + push + link)   [default]
claude-sync push         commit and push local changes
claude-sync pull         pull remote changes + link
claude-sync link         set up / refresh symlinks
claude-sync unlink       remove repo-owned symlinks (keeps the repo)
claude-sync clean        remove broken symlinks from ~/.claude/projects
claude-sync status       show repo, projects, link health, and changes
claude-sync help         show usage
```

## Configuration

| Env var | Meaning |
|---|---|
| `CLAUDE_SYNC_DIR` | Path to your private config repo. Required for all commands except `init` (unless the script lives inside the data repo, for backward compatibility). |
| `CLAUDE_SYNC_WORKDIR` | Parent directory under `$HOME` to strip from canonical project names. e.g. `develop` strips `~/develop/` from paths. Per-device. |
| `CLAUDE_SYNC_REMOTE` | Git remote name (default `origin`). |

The branch your data repo is currently on is used for push/pull, so `main`, `master`, or any other branch works.

### Workdir example

Say all your projects live under `~/develop/`. Without `CLAUDE_SYNC_WORKDIR`, canonical names look like `-develop-myapp`. With `export CLAUDE_SYNC_WORKDIR=develop`, they collapse to `-myapp`, and a colleague whose projects live under `~/code/` can set `CLAUDE_SYNC_WORKDIR=code` to share canonical names.

## Privacy

Your data repo will contain:

- Your `settings.json` — **this may include MCP server tokens or other secrets** depending on how you've configured Claude Code.
- Project memory files with your work context.
- Custom skills and CLAUDE.md instructions.

**Use a private repo.** This is your responsibility — `claude-sync` does not encrypt or redact anything.

## Uninstall

```bash
claude-sync unlink                    # remove only repo-owned symlinks from ~/.claude/
rm -rf ~/.local/share/claude-sync     # remove the tool
rm ~/.local/bin/claude-sync           # remove the bin symlink
```

Your data repo at `$CLAUDE_SYNC_DIR` is left intact.

## Contributing

The tool is a single bash script in `bin/claude-sync`. Run `shellcheck bin/claude-sync install.sh` before sending changes. Issues and PRs welcome.

## License

[MIT](LICENSE)
