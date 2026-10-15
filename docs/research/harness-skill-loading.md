# How Claude Code, Codex and Cursor load skills

Answers Chr1sC0der/skillbag#2. Researched 2026-10-15 from official docs only; each claim cites its owner. Quotes are from the fetched pages; where a page is silent this is stated rather than inferred.

Sources:
- Claude Code: <https://code.claude.com/docs/en/skills>
- Codex: <https://developers.openai.com/codex/skills> (308-redirects to <https://learn.chatgpt.com/docs/build-skills>)
- Cursor: <https://cursor.com/docs/context/skills>
- Agent Skills spec (shared open format): <https://agentskills.io/specification>

## Summary

| | Claude Code | Codex | Cursor |
|---|---|---|---|
| Native `SKILL.md` folders | Yes | Yes | Yes |
| Global (user) path | `~/.claude/skills/<name>/` | `$HOME/.agents/skills` | `~/.agents/skills/` or `~/.cursor/skills/` |
| Project path | `.claude/skills/<name>/` | `.agents/skills` (cwd up to repo root) | `.agents/skills/` or `.cursor/skills/` |
| Reads other harness dirs | No | No (documented) | Yes: `.claude/skills`, `.codex/skills`, `~/.claude/skills`, `~/.codex/skills` |
| Follows symlinks | Yes (skill dir entries); plugins differ | Yes | Not documented |
| `name` vs directory | `name` is optional, defaults to directory; either invokes it | `name` required; no directory rule documented | `name` required; must match parent folder |
| `name` characters | No pattern documented; only unsafe chars refused | Not documented | Lowercase letters, numbers, hyphens |
| Namespacing | Plugin skills are `plugin:skill`; nested clashes are `path:skill` | None documented; duplicate names both listed | None; folder is identity, category folders organisational |

The Agent Skills spec (agentskills.io) is the common baseline: `name` is 1-64 chars, lowercase alphanumerics and hyphens, no leading/trailing/consecutive hyphens, and must match the parent directory. Codex and Cursor pages describe the same format; Claude Code is looser.

## Claude Code

- Locations: enterprise (managed settings dir), personal `~/.claude/skills/<skill-name>/SKILL.md`, project `.claude/skills/<skill-name>/SKILL.md`, nested `<subdir>/.claude/skills/...`, `--add-dir` directories, plugin `<plugin>/skills/<skill-name>/SKILL.md`, and claude.ai account skills. Legacy `.claude/commands/` still works.
- Discovery: project skills load from `.claude/skills/` in the start directory and every parent up to the repo root. Nested skills below the start directory load the first time Claude reads or edits a file there. In a linked git worktree, parent search stops at the worktree root; from v2.1.277, if the worktree has no root `.claude/skills`, the main checkout's project skills load instead.
- Format: YAML frontmatter plus Markdown. "All fields are optional. Only `description` is recommended." Unknown fields are ignored; unparseable YAML still loads the skill with no fields set.
- Symlinks: "A `<skill-name>` entry in the enterprise, personal, or project location can be a symlink to a directory elsewhere on disk. Claude Code reads `SKILL.md` from the target and loads the skill once even if several locations point at the same target. Plugin skills handle symlinks differently." (Docs do not say how.)
- Names: `name` "Defaults to the directory name." In personal/project dirs `name` sets the `/` command, and "the directory name also invokes the skill". No allowed-character pattern or max length is documented (the 64-char/lowercase rule is from the Agent Skills spec, not this page). Refused: zero-width/control/private-use/direction-changing characters, odd blanks, `<` `>` in a folder or file name (such skills do not load; in frontmatter `name` they are accepted but do not invoke). Reserved: `anthropic-skills` (outside plugins) and `synced` as folder names.
- Namespacing: plugin skills are `/plugin-name:skill-name`; plugin `name` sets only the last segment. Nested skills with clashing names use the subdirectory path, e.g. `/apps/web:deploy`.
- Precedence: enterprise over personal over project. Plugin skills never collide with local ones because they are namespaced. Project-root and nested skills both load.
- Live reload: skill dirs are watched; a newly created top-level skills dir needs `/reload-skills`.

## Codex

- Locations: REPO `$CWD/.agents/skills`, `$CWD/../.agents/skills`, `$REPO_ROOT/.agents/skills` (scans cwd up to repo root); USER `$HOME/.agents/skills`; ADMIN `/etc/codex/skills`; SYSTEM bundled. Note: this is `.agents/skills`, not `.codex/skills`.
- Format: a directory with `SKILL.md` (plus optional `scripts/`, `references/`, `assets/`, `agents/openai.yaml`). Frontmatter must include `name` and `description`.
- Symlinks: "Codex supports symlinked skill folders and follows the symlink target when scanning these locations."
- Names: the page states no character rule, length limit, or relation of `name` to directory name. Duplicates: "If two skills share the same `name`, Codex doesn't merge them; both can appear in skill selectors." No precedence or namespacing scheme is described.
- Control: `[[skills.config]]` with `path` and `enabled = false` in `~/.codex/config.toml` (restart needed); `allow_implicit_invocation: false` in `agents/openai.yaml` keeps explicit `$skill` invocation only.

## Cursor

- Locations: project `.agents/skills/` and `.cursor/skills/`; user `~/.agents/skills/` and `~/.cursor/skills/`. Only `~/.cursor/skills/` syncs to Cloud Agents. "For compatibility" Cursor also loads `.claude/skills/`, `.codex/skills/`, `~/.claude/skills/`, `~/.codex/skills/`. Nested project subdirectories are discovered and scoped to files inside that directory.
- Format: folder containing `SKILL.md`; required frontmatter `name`, `description`; optional `paths`, `disable-model-invocation`, `icon`, `color`, `metadata`; optional `scripts/`, `references/`, `assets/`.
- Names: "Lowercase letters, numbers, and hyphens only." and "Must match the parent folder name." No max length stated.
- Symlinks: the page does not address them. Unverified; test empirically before relying on them.
- Namespacing: none. "The category folder is purely organizational"; identity is the folder holding `SKILL.md`.
- Native equivalent: Cursor rules. Migration converts "Apply Intelligently" rules (no `globs`, `alwaysApply` false/undefined) to skills and slash commands to skills with `disable-model-invocation: true`; `alwaysApply: true` and glob-scoped rules, and user rules, are not migrated. The page gives no `.cursor/rules` path.

## Implications for skillbag

- One physical skill folder can serve all three if its `name` is lowercase-hyphen and equals its directory name (satisfies Cursor and the spec; Claude Code and Codex accept it).
- `~/.agents/skills` is read natively by both Codex and Cursor; Claude Code reads only `~/.claude/skills`. Cursor additionally reads `~/.claude/skills`, so a symlink farm into `~/.claude/skills` plus `~/.agents/skills` covers all three.
- Symlink-installed skills are documented for Claude Code and Codex; Cursor is the open risk.

## Open gaps

- Cursor symlink behavior; Codex `name` constraints and precedence between REPO/USER/ADMIN; how Claude Code plugin symlinks differ.
