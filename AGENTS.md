# Power Platform Skills - Development Guidelines

This file provides guidance to AI Agents when working with code in this repository.

## What This Repo Is

A **plugin marketplace** for Power Platform development by Microsoft. The Open Plugins marketplace manifest (`marketplace.json`) references individual plugins in `plugins/`. Legacy `.claude-plugin` manifest mirrors are kept for existing subscriptions. Each plugin has its own `AGENTS.md` with plugin-specific guidance.

## Repository Structure

```
power-platform-skills/
├── marketplace.json          # Open Plugins marketplace manifest (lists all available plugins)
├── .claude-plugin/           # Legacy manifest mirrors for existing subscriptions
│   └── marketplace.json
├── plugins/                  # Directory containing individual plugins
│   └── <plugin-name>/        # Individual plugin (e.g., power-pages)
│       ├── .plugin/
│       │   └── plugin.json   # Open Plugins manifest
│       ├── .claude-plugin/
│       │   └── plugin.json   # Legacy manifest mirror
│       ├── AGENTS.md         # Plugin-specific development guidelines
│       ├── agents/           # Agent persona files
│       ├── commands/         # Command entry points
│       ├── shared/           # Shared resources and documentation
│       └── skills/           # Skill workflows (SKILL.md in subdirectories)
├── shared/                   # Cross-plugin shared resources
│   └── skills/               # Shared skill definitions
│       └── <skill-name>/     # SKILL.template.md + workflow .md files
├── AGENTS.md                 # Generic development guidelines (this file)
└── README.md                 # Repository overview
```

## Local Development

Test a plugin locally by launching your AI agent with the plugin path:

```bash
claude --plugin-dir /path/to/plugins/<plugin-name>
```

No root-level build, lint, or test commands exist. Build/test tooling lives inside each plugin.

## Plugin Conventions

Each plugin follows this structure:

- `.plugin/plugin.json` — Open Plugins metadata (name, version, keywords)
- `.claude-plugin/plugin.json` — legacy mirror of `.plugin/plugin.json` kept for existing subscriptions
- `.mcp.json` — MCP server configuration (optional)
- `agents/` — Agent definitions (`.md` files with YAML frontmatter)
- `skills/` — Skill definitions, each in its own subdirectory with a `SKILL.md`
- `scripts/` — Shared utility scripts referenced by skills and agents
- `references/` — Shared reference documents used by multiple skills

Skills are defined in `SKILL.md` files with YAML frontmatter (name, description, allowed-tools, model, hooks). The `allowed-tools` field must use a **comma-separated list** (e.g., `allowed-tools: Read, Write, Edit, Bash, Glob, Grep`) — not JSON array syntax (`["Read", "Write"]`) or YAML list syntax. Each skill may include validation scripts in a `scripts/` subdirectory, run as Stop hooks when the skill session ends.

## Equinor Alignment Work

When aligning this fork for internal Equinor use, start with [docs/equinor-alignment/README.md](docs/equinor-alignment/README.md). The baseline and review checklist define the governance, Tech Radar, MCP, EDS, Power Platform zone, and publication requirements that must be checked before plugin metadata, skills, scripts, or manifests are promoted internally.

Do not mark a plugin as ready for internal publication until it has a completed review record that conforms to `docs/equinor-alignment/plugin-review.schema.json` and any required Power Platform owner review has been captured.

## Cross-Plugin Shared Skills

Skills that apply to all plugins live in `shared/skills/<skill-name>/`. The workflow logic is written once in a shared `.md` file, and each plugin has a thin `skills/<skill-name>/SKILL.md` that contains only the YAML frontmatter and a reference to the workflow path bundled inside that plugin at install time.

**Pattern:**

- `shared/skills/<skill-name>/<workflow>.md` — Full workflow (phases, instructions, field definitions)
- `shared/skills/<skill-name>/SKILL.template.md` — Template SKILL.md (frontmatter + reference to workflow); supports `{{PLUGIN_NAME}}` placeholder
- `plugins/<plugin>/skills/<skill-name>/SKILL.md` — Per-plugin wrapper generated from the template above
- `plugins/<plugin>/skills/<skill-name>/<workflow>.md` — Bundled workflow file copied into the plugin so installs work without repo-root shared paths

This keeps the skill discoverable in each plugin while preserving install-time portability. Marketplace installs copy only the plugin directory, so per-plugin wrappers must not reference repo-root `shared/` paths at runtime. Instead, point the wrapper at `${PLUGIN_ROOT}/skills/<skill-name>/<workflow>.md` and keep a physical copy of the shared workflow at that per-plugin path. Do not use Git symlinks for shared content; Windows and plugin-host installs can materialize them as plain link files. When updating a shared skill, edit the workflow file and/or `SKILL.template.md` in `shared/`, then refresh the per-plugin wrappers (frontmatter + bundled workflow reference, with `{{PLUGIN_NAME}}` substituted) and copy the workflow content into each adopting plugin. Commit the shared source and per-plugin copies together.

## Legacy Marketplace Compatibility

Keep the root `.claude-plugin/marketplace.json` and each plugin's `.claude-plugin/plugin.json` as JSON mirrors of their Open Plugins counterparts. The shared root marketplace must stay dual-compatible while keeping per-plugin entries minimal: each plugin entry should include only the required `name` and repository-root-relative `source` fields. Keep marketplace-level `owner` and `metadata` because they describe the collection, but store per-plugin display/update metadata (description, version, license, keywords, and similar fields) in each `.plugin/plugin.json` instead of duplicating or overriding it in the marketplace index. Existing marketplace subscriptions may still resolve the legacy paths during auto-update, so removing or drifting these files can force users to reinstall. Because mirrors are committed files, update both source and legacy copies together, then run `node scripts/validate-legacy-compatibility.js` after metadata changes.

## Code Conventions

**DRY (Don't Repeat Yourself):** Never duplicate logic across files. Each plugin has shared utilities (e.g., `scripts/lib/`) and shared reference docs (e.g., `references/`). Always check for and reuse existing helpers before writing new code. When adding shared logic, put it in the plugin's shared modules — not in individual skill directories.

### Code comments

Most code in this repo is Node.js scripts and hooks that shell out to `pac`/`az`, call the Dataverse and Power Platform APIs, and parse loosely structured CLI output. The reasoning behind a line is rarely obvious from the line alone, so comments matter.

- Err on the side of over-commenting when the reasoning is not obvious. Comments should explain **why** code is written a particular way.
- Comment non-obvious implementation details: concurrency hazards, lifecycle constraints, compatibility requirements, platform quirks, upstream PAC CLI or Dataverse workarounds, and intentional deviations from the obvious helper or API.
- When parsing strings, logs, CLI output, OData payloads, or other loosely structured data, include a comment with an example of the raw format being parsed.
- When code follows an external standard, protocol, or Power Platform convention, include links to the relevant documentation so future readers can verify the rule.
- When code touches auth tokens or other privacy/security-sensitive flows, explain the scope and fail-closed behavior.
- Do not add comments that only narrate clear code.

## Maintaining This File

When you add new plugins or change the repository-level structure, update this file. For plugin-specific changes, update the plugin's own `AGENTS.md` (e.g., `plugins/power-pages/AGENTS.md`).

## External Documentation

- [Power Pages Code Sites](https://learn.microsoft.com/en-us/power-pages/configure/create-code-sites)
- [PAC CLI Reference](https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/pages)
- [Create Website API](https://learn.microsoft.com/en-us/rest/api/power-platform/powerpages/websites/create-website)
