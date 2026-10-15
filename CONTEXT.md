# skillbag

Manages skill libraries across AI harnesses, so a user's skills travel between projects and tools and can be enabled or disabled together or individually.

## Language

**Skill**:
A folder containing a `SKILL.md` (frontmatter plus instructions) and optional supporting files. The canonical unit skillbag manages.
_Avoid_: Rule, instruction file, prompt

**Skill library**:
A named, versioned collection of skills with a source, such as a git repo or a local folder. skillbag consumes and enables libraries; it does not author them.
_Avoid_: Pack, bundle, plugin, collection

**Harness**:
An AI coding tool that loads skills, in its own native shape. v1 targets Claude Code, Codex, and Cursor.
_Avoid_: Tool, client, agent, IDE

**Adapter**:
The piece that places a skill where a harness loads it, or skips it with a warning when the target cannot accept it. One adapter can serve several harnesses that read the same location.
_Avoid_: Driver, plugin, connector

**Enable / Disable**:
Making a skill or a whole skill library available to a harness, or withdrawing it, at global or per-project scope.
_Avoid_: Install/uninstall, activate

**Install**:
Adding a skill library to the store. Installing a library enables all its skills globally by default.
_Avoid_: Add, import, download

**Project snapshot**:
A project's own copy of the global enabled set, taken when the project is created. Later global changes never reach it; per-project disables are remembered only in the project.
_Avoid_: Inheritance, override layer

**Skill reference**:
A skill naming another skill it depends on (Wayfinder references Grilling and Research). Enabling a skill also enables every skill it references, transitively.
_Avoid_: Dependency, import, nested skill

**Namespaced skill**:
A skill addressed by its library plus its name (`libA:tdd`), so same-named skills from different libraries coexist. skillbag warns the user when enabled libraries ship the same skill name, but does not block.
_Avoid_: Collision, conflict, alias

**Store**:
The user-level location holding the user's skill libraries, from which skills are enabled per project or globally.
_Avoid_: Registry, cache, repository
