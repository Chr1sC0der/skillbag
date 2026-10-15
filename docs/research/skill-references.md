# What do existing skill libraries declare about references between skills?

Resolves issue Chr1sC0der/skillbag#3. Researched 2026-10-15.

## Answer

**Nobody declares skill-to-skill dependencies in frontmatter. References live only in prose.** No format, library, or doc surveyed has a SKILL.md field that says "this skill needs skill X".

- The Agent Skills spec has no such field; Claude Code's frontmatter has none either and silently ignores unknown fields.
- The only machine-readable dependency mechanisms are at other levels: `dependencies` in `plugin.json` (plugin-to-plugin), and `skills` in a subagent definition (subagent-to-skill preload).
- In the two libraries inspected, prose references take two conventions:
  - mattpocock-skills: imperative `call the Skill tool with "<name>"` using bare skill names.
  - superpowers: `superpowers:<name>` qualified names, marked `**REQUIRED SUB-SKILL:**` or `**REQUIRED BACKGROUND:**`.
- skillbag will have to **detect references by scanning body text** (and record them in its own metadata). There is no declared source of truth to read.

## Evidence

### 1. Specs and official docs (primary)

| Source | Finding |
| - | - |
| Agent Skills spec, https://agentskills.io/specification | Frontmatter fields are exactly `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`. No dependency field. `compatibility` is only for "intended product, system packages, network access". `metadata` is an "arbitrary key-value mapping" (string to string) which clients may use for properties the spec does not define, so a dependency list could be stored there, but nothing standardises it. The spec's only inter-file referencing guidance is relative file paths, kept "one level deep from SKILL.md". |
| Claude Code skills docs, https://code.claude.com/docs/en/skills | 20 supported fields: `name, description, when_to_use, argument-hint, arguments, disable-model-invocation, user-invocable, allowed-tools, disallowed-tools, model, effort, context, agent, background, hooks, paths, shell, metadata, license, compatibility`. None declares or preloads other skills. Quote: "Claude Code ignores a field it doesn't recognize without reporting an error." (`agent` picks a subagent type for forked skills; it does not load skills.) |
| Claude Code subagents docs, https://code.claude.com/docs/en/sub-agents | The `skills` field exists only in **subagent definitions**: "Skills to preload into the subagent's context at startup. The full skill content is injected... Subagents can still invoke unlisted project, user, and plugin skills through the Skill tool." So it is a consumer-side preload list, not a dependency declared by a skill. |
| Claude Code plugin manifest, https://code.claude.com/docs/en/plugins-reference and https://code.claude.com/docs/en/plugins/dependencies | `plugin.json` has `dependencies`: "Plugins that must be enabled for this one to work." Entries are `"name"`, `"name@marketplace"`, or `{name, marketplace, version}` with a semver range, resolved against git tags `<plugin>--v<version>`. The granularity is the **plugin**, not the skill. |

Also noted: skills inside a plugin are namespaced `/plugin-name:skill-name` (skills docs), which is the same `plugin:skill` form superpowers uses in prose.

### 2. Local survey of installed libraries

Scanned all 99 `SKILL.md` files under `~/.claude/plugins/cache/` and `~/.claude/plugins/marketplaces/` (mattpocock-skills 1.2.3, superpowers 6.4.2, chrome-devtools-mcp 1.9.0, plugin marketplaces).

Frontmatter keys actually in use (count of skills): `name` 99, `description` 99, `disable-model-invocation` 25, `version` 13, `allowed-tools` 10, `user-invocable` 8, `argument-hint` 7, `tools` 2, `disallowed-tools` 2, `metadata` 1, `license` 1. **Zero dependency/requires/skills fields.**

Plugin manifests: mattpocock-skills' `plugin.json` has only metadata and a `skills` array of paths (which enumerates the shipped skills, not their dependencies). superpowers' `plugin.json` has no `dependencies`. A grep for "dependenc" in the plugin JSON manifests found only npm `package.json` files and unrelated marketplace descriptions.

#### mattpocock-skills: imperative "call the Skill tool" with bare names

Bare name, no plugin prefix, dispatched at runtime. Concrete examples (paths under `~/.claude/plugins/cache/claude-plugins-official/mattpocock-skills/1.2.3/skills/`):

- `engineering/wayfinder/SKILL.md` (the example named in the issue):
  - line 77: Research tickets are "Resolved by a subagent that calls the Skill tool with "research""
  - line 78: Prototype tickets "call[] the Skill tool with "prototype""
  - line 79: Grilling tickets: "Always call the Skill tool twice, for "grilling" and "domain-modeling""
  - line 111: "Call the Skill tool twice, for "grilling" and "domain-modeling"" to name the destination
  - line 124: "call the Skill tool for whichever skills the `## Notes` block names" (a **data-driven** reference: skill names stored in ticket text, not in the skill)
- `productivity/grill-me/SKILL.md` line 7: `Call the Skill tool with "grilling".` (thin wrapper)
- `engineering/grill-with-docs/SKILL.md` line 7: `Call the Skill tool twice, for "grilling" and "domain-modeling".`
- `engineering/triage/SKILL.md` line 76: grill step calls "grilling" and "domain-modeling".
- `engineering/improve-codebase-architecture/SKILL.md` lines 13, 64, 66, 71: references "codebase-design", "grilling", "domain-modeling" (some conditional on a branch of the flow).
- `engineering/tdd/SKILL.md` line 26: conditionally calls "codebase-design".
- `in-progress/retro/SKILL.md`, `in-progress/setup-ts-deep-modules/SKILL.md`: "writing-for-agents", "codebase-design".
- `productivity/handoff/SKILL.md` line 10 and `in-progress/claude-handoff`: tell the *next* agent which skills to call (references emitted into an artifact, not a dependency).

Pocock's own `writing-for-agents` skill names the underlying concept a **context pointer**: "a reference held in the agent's context that names some out-of-context material and encodes the condition for reaching it... The pointer's wording, not its target, decides when the agent reaches the material." Skill-to-skill references are treated as pointers whose reliability depends on prose wording, not as declared edges.

Pocock's repo `AGENTS.md` / `CLAUDE.md` governs registration (every promoted skill needs a README link and a `plugin.json` `skills` entry) but says nothing about declaring inter-skill dependencies.

#### superpowers: qualified names with a REQUIRED marker

Paths under `~/.claude/plugins/cache/anthropic-plugin-directory/superpowers/6.4.2-8ca22dba9a94/skills/`:

- `writing-skills/SKILL.md` section "5. Cross-Referencing Other Skills" (lines ~278-288) is the only place any library documents a convention:
  - Good: `**REQUIRED SUB-SKILL:** Use superpowers:test-driven-development`
  - Good: `**REQUIRED BACKGROUND:** You MUST understand superpowers:systematic-debugging`
  - Bad: `See skills/testing/test-driven-development` ("unclear if required")
  - Bad: `@skills/testing/test-driven-development/SKILL.md` ("force-loads, burns context")
  - It also says the spec at agentskills.io is the reference for supported frontmatter fields (line 96).
- `writing-skills/SKILL.md` line 18: `**REQUIRED BACKGROUND:** You MUST understand superpowers:test-driven-development before using this skill.`
- `writing-plans/SKILL.md` lines 59, 201, 204: `REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans`. This is an either/or edge.
- `executing-plans/SKILL.md`: lines 149, 219 `REQUIRED SUB-SKILL`; plain references at 47, 53, 111, 195, 242, 304; also inside Graphviz node labels (lines 88, 104) as `Use superpowers:finishing-a-development-branch`.
- `systematic-debugging/SKILL.md` lines 177, 189: "Use the `superpowers:test-driven-development` skill", "`superpowers:verification-before-completion`".
- `using-superpowers/SKILL.md` lines 30-31: routing table "Fix this bug" -> `superpowers:systematic-debugging` first.
- `using-superpowers/references/gemini-tools.md`: per-harness mapping of these references (shows the same prose reference needs translating across harnesses).

All frontmatter in superpowers is just `name` and `description`.

## Implications for skillbag

1. **Detection must be textual.** No declared edges exist to read. A scanner has to find references in `SKILL.md` bodies (and linked sibling files).
2. **Patterns to recognise** (from the evidence above):
   - `call the Skill tool with "<name>"`, including `twice, for "<a>" and "<b>"` (multiple names in one sentence).
   - `<plugin>:<skill>` qualified names (e.g. `superpowers:writing-plans`), including inside backticks, Graphviz labels, and `REQUIRED SUB-SKILL` / `REQUIRED BACKGROUND` lines.
   - Plain relative links (`[x](../other/SKILL.md)`) are the spec-blessed form but were **not found** in the mattpocock-skills or superpowers skill bodies (a targeted grep for sibling `SKILL.md` links in mattpocock-skills returned nothing), so they are rarely used for skill-to-skill edges.
3. **Edge strength is only in the wording.** Superpowers' `REQUIRED` vs plain "use" is the lone strength signal; Pocock's references carry conditions ("if the user picks a candidate", "when the shape is in question"). A detector can capture the target and a coarse required/optional flag, but not the condition reliably.
4. **Some references are not static dependencies.** Wayfinder step 124 reads skill names from ticket data at runtime; handoff skills emit skill names for another agent. A static scan will see only literal names; treat these as "dynamic/unresolved" rather than missing.
5. **Names are not globally unique.** Pocock uses bare names; superpowers uses `plugin:skill`. skillbag should resolve a bare name against the same library first and keep the plugin qualifier when present.
6. **Where to record them.** Since the Agent Skills spec offers `metadata` (string-to-string, "clients can use this to store additional properties"), and Claude Code ignores unknown fields, skillbag could write detected refs there or, more safely, keep them in its own lockfile/manifest outside `SKILL.md`. The latter avoids editing third-party skill files. For bundling, the only official dependency vocabulary is plugin-level `dependencies` with semver ranges (plugin granularity), which skillbag can mirror conceptually but cannot lean on for skill-level edges.
7. **Cross-harness caveat:** the subagent `skills` preload list and plugin `dependencies` are Claude Code features; the Agent Skills spec itself defines neither, so portable behaviour has to come from skillbag's own record.

## Limits of this research

- Local survey covers only libraries installed on this machine (mattpocock-skills 1.2.3, superpowers 6.4.2, chrome-devtools-mcp, official plugin marketplace). The Anthropic first-party `anthropics/skills` repo was not read directly; its conventions are inferred from the spec and the bundled `skill-creator` / `plugin-dev` skills, which document no dependency field.
- The Claude Code sub-agents page was read only up to the first 100,000 characters; the quoted `skills` field sections fell within that range.
