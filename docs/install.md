# Install guide

Everything here was checked against OpenCode **2.0.6** on macOS. Where a behavior was observed rather than documented, it says so.

## Requirements

- OpenCode V2 with a reachable service (`opencode service status`).
- A V2 CLI on your `PATH`. This guide writes `opencode`. Some installs expose the V2 binary under another name, such as `opencode2` — substitute yours throughout. The executable name alone does not identify the version, so confirm with `--version`.
- `git` for the clone-based options, or `curl` for the file-based one.
- `python3` is optional. The skill uses it only for the quoted-heredoc pattern when message text is long or contains awkward quoting.

```bash
opencode --version        # expect v2.0.x
opencode service status
```

## Where OpenCode looks for skills

| Scope | Directory | Status |
|---|---|---|
| Global | `~/.config/opencode/skills` | canonical |
| Global | `~/.claude/skills`, `~/.agents/skills` | compatibility |
| Project | `.opencode/skills` | canonical |
| Project | `.claude/skills`, `.agents/skills` | compatibility |

Project directories are searched from the current directory up to the project root, and every matching directory along the way contributes.

You can add more sources — local directories or HTTP catalogs — with the `skills` array in any `opencode.json` / `opencode.jsonc`. Entries from all discovered config files are combined rather than replaced.

## How the skill ID is derived

This matters, because the agent loads the skill by ID, and the ID comes from the filesystem — not from the `name` field in the frontmatter, which is only a display label.

Observed on 2.0.6:

| Layout under a skill source root | Resulting ID |
|---|---|
| `session-messaging/SKILL.md` | `session-messaging` |
| `group/nested/session-messaging/SKILL.md` | `session-messaging` — only the immediate parent directory counts, at any depth |
| `session-messaging.md` (flat file at the source root) | `session-messaging` |
| a symlink named `session-messaging` → a directory containing `SKILL.md` | `session-messaging` — symlinks are followed |
| `SKILL.md` placed directly in the source root | the **source root's own name** (e.g. `skills`) — avoid this |

Practical rule: keep `SKILL.md` inside a directory named exactly `session-messaging`, and the ID is correct no matter which install option you pick.

## Option A — clone anywhere, symlink (recommended)

```bash
git clone --depth 1 https://github.com/TheFerragamo/opencode-session-messaging.git ~/src/opencode-session-messaging
mkdir -p ~/.config/opencode/skills
ln -s ~/src/opencode-session-messaging/skills/session-messaging ~/.config/opencode/skills/session-messaging
```

Repo stays out of your config directory; `git -C ~/src/opencode-session-messaging pull` updates the skill in place.

## Option B — clone anywhere, register the source directory

```bash
git clone --depth 1 https://github.com/TheFerragamo/opencode-session-messaging.git ~/src/opencode-session-messaging
```

Add the repo's `skills/` directory to `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": ["~/src/opencode-session-messaging/skills"]
}
```

The registered directory becomes a source root, so `skills/session-messaging/SKILL.md` gets the ID `session-messaging`. If you already have a `skills` array, append to it.

Path resolution rules for that array: `~/` expands to your home directory, absolute paths are used as written, and a **relative path resolves from OpenCode's active working directory, not from the config file** — which is why this guide uses `~/`.

## Option C — no git, two files

```bash
BASE=https://raw.githubusercontent.com/TheFerragamo/opencode-session-messaging/main/skills/session-messaging
DEST=~/.config/opencode/skills/session-messaging
mkdir -p "$DEST/references"
curl -fsSL "$BASE/SKILL.md" -o "$DEST/SKILL.md"
curl -fsSL "$BASE/references/verification.md" -o "$DEST/references/verification.md"
```

`SKILL.md` links to `references/verification.md` relatively, so keep that layout. Updating means re-running the two `curl` commands.

## Option D — project-scoped install

To make the skill available only inside one project, put it under that project's `.opencode/skills/` instead:

```bash
git clone --depth 1 https://github.com/TheFerragamo/opencode-session-messaging.git /tmp/sm
mkdir -p .opencode/skills
cp -R /tmp/sm/skills/session-messaging .opencode/skills/session-messaging
rm -rf /tmp/sm
```

Commit it if your team should share it; add it to `.gitignore` if it is yours alone.

## Verify

```bash
opencode api get /api/skill
```

Expect an entry like:

```json
{
  "id": "session-messaging",
  "name": "Session Messaging",
  "description": "Exchange messages with agents in other OpenCode V2 sessions …",
  "path": "/Users/you/.config/opencode/skills/session-messaging/SKILL.md"
}
```

Check the `path` — it tells you which copy is actually live.

To list skills as seen from a specific project directory, pass the location (the directory must be a recognized project, e.g. a git repository):

```bash
opencode api get "/api/skill?location%5Bdirectory%5D=$PWD"
```

## Update

```bash
# Options A and B
git -C ~/src/opencode-session-messaging pull

# Option C
# re-run the two curl commands
```

## Uninstall

```bash
# Option A
rm ~/.config/opencode/skills/session-messaging          # removes the symlink only
rm -rf ~/src/opencode-session-messaging                 # removes the clone

# Option B — delete the entry from the "skills" array, then remove the clone

# Option C / D
rm -rf ~/.config/opencode/skills/session-messaging
rm -rf .opencode/skills/session-messaging
```

## Troubleshooting

**The skill is not in `/api/skill` yet.** The list is cached for a short time. It refreshes on its own — wait a moment and query again. If you have no session mid-run, `opencode service restart` forces it; this interrupts anything currently executing, so check `opencode api get /api/session/active` first.

**Two copies, and the wrong one is live.** When one ID exists in several skill directories, OpenCode keeps a single copy and silently drops the others. Observed precedence: canonical beats compatibility — `.opencode/skills` over `.claude/skills`, and `~/.config/opencode/skills` over `~/.agents/skills`. Delete the stale copy instead of relying on the ordering, and confirm with the `path` field.

**The ID came out as `skills` (or something else unexpected).** `SKILL.md` ended up directly in a source root. Move it into a directory named `session-messaging`.

**The agent doesn't load it.** Skills are gated by permissions using the `skill` action with ID patterns, where the last matching rule wins. A `deny` rule hides the skill entirely. The skill also needs your agent's `shell` tool to reach the service API — an `allowed-tools` frontmatter field does **not** grant or enforce access in V2.

**`--param` seems to be ignored.** On CLI 2.0.3 and 2.0.6, `--param` works with operation IDs but is silently ignored when you pass a raw `METHOD /path`. Put query parameters in the quoted path instead — the skill already does this everywhere.

**`--data @file.json` sends the literal string.** Not supported in either tested build; the CLI passes the argument straight through as the HTTP body. Pass JSON inline, or build it in memory. A `SessionNotFoundError` from a bad recipient ID does not prove the body parsed.

## Using it from other agents

`SKILL.md` is a portable markdown skill, so other agents that read the same format can load it from their own skill directory (for example, Claude Code reads `~/.claude/skills`). The protocol itself only requires a shell tool and the `opencode2` CLI — it does not require the agent to be running inside OpenCode. Directory naming conventions differ between tools, so keep the directory named `session-messaging`.
