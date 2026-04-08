# agentsys

> A modular runtime and orchestration system for AI agents - works with Claude Code, OpenCode, Codex CLI, Cursor, and Kiro

## Critical Rules

1. **Plain text output** - No emojis, no ASCII art. Use `[OK]`, `[ERROR]`, `[WARN]`, `[CRITICAL]` for status markers.
2. **No unnecessary files** - Don't create summary files, plan files, audit files, or temp docs.
3. **Task is not done until tests pass** - Every feature/fix must have quality tests.
4. **Create PRs for non-trivial changes** - No direct pushes to main.
5. **Always run git hooks** - Never bypass pre-commit or pre-push hooks.
6. **Use single dash for em-dashes** - In prose, use ` - ` (single dash with spaces), never ` -- `.
7. **Report script failures before manual fallback** - Never silently bypass broken tooling.
8. **Token efficiency** - Save tokens over decorations.

## Model Selection

| Model | When to Use |
|-------|-------------|
| **Opus** | Complex reasoning, analysis, planning |
| **Sonnet** | Validation, pattern matching, most agents |
| **Haiku** | Mechanical execution, no judgment needed |

## Core Priorities

1. User DX (plugin users first)
2. Worry-free automation
3. Token efficiency
4. Quality output
5. Simplicity

## Repository Structure

```
agentsys/
├── bin/                    # CLI entry points
│   ├── cli.js              # npm installer (multi-platform, ~80KB)
│   └── dev-cli.js          # Developer command dispatcher (~19KB)
├── lib/                    # Core library (@agentsys/lib workspace package)
│   ├── platform/           # Platform detection (Claude, OpenCode, Codex, Cursor, Kiro)
│   ├── patterns/           # slop-patterns.js, review-patterns.js, pipeline, cli-enhancers
│   ├── enhance/            # 25+ analyzers: agent, prompt, plugin, skill, hook, security, cross-file
│   ├── collectors/         # Data collection: codebase, docs, github, git analysis
│   ├── state/              # Workflow state management
│   ├── config/             # Configuration management
│   ├── perf/               # Performance analysis tools
│   ├── drift-detect/       # Documentation vs. code comparison
│   ├── repo-map/           # Repository mapping and intelligence
│   ├── binary/             # Binary utilities
│   ├── sources/            # custom-handler, policy-questions, source-cache
│   ├── discovery/          # Plugin/agent/skill auto-discovery
│   ├── utils/              # context-optimizer, shell-escape, command-parser
│   ├── cross-platform/     # Cross-platform compatibility layer
│   ├── types/              # TypeScript definitions (.d.ts)
│   └── schemas/            # Data schemas
├── __tests__/              # Jest test suite (72 files)
├── tests/                  # Integration/module tests (12 files)
├── adapters/               # Platform adapters: codex, opencode, opencode-plugin
├── site/                   # Static website (hardcoded HTML + content.json)
├── scripts/                # Dev utilities: validate-*, generate-*, scaffold, bump-version
├── docs/                   # Architecture, testing, usage, cross-platform docs
├── checklists/             # Step-by-step action checklists
├── .github/workflows/      # CI/CD: ci.yml, release.yml, claude-code-review.yml, deploy-site.yml
├── .claude/                # Claude Code: settings.json, hooks/
├── .opencode/              # OpenCode platform files
├── .codex/                 # Codex CLI platform files
└── .kiro/                  # Kiro platform files
```

## Dev Commands

```bash
npm test                          # Run all tests (84 files, sequential)
npm run validate                  # All validators
npm run validate:plugins          # Plugin structure validation
npm run validate:cross-platform   # Cross-platform compatibility check
npm run validate:consistency      # Version consistency across files
npm run validate:counts           # Site count accuracy
npm run gen-docs                  # Regenerate documentation
npm run gen-adapters              # Regenerate platform adapters
npm run expand-templates          # Expand agent templates
npm run new:plugin                # Scaffold a new plugin
npm run new:agent                 # Scaffold a new agent
npm run new:skill                 # Scaffold a new skill
npm run new:command               # Scaffold a new command
```

## Testing

- Framework: Jest 29, Node environment, `maxWorkers: 1` (sequential - shared filesystem state)
- Coverage threshold: 10% (statements, lines, functions, branches)
- Import alias: `@agentsys/lib` maps to `lib/index.js`
- Tests ignore: `plugins/*/lib/`, `.claude/worktrees/`, `worktrees/`
- Run with coverage: `npm run test:coverage`

Every feature/fix requires tests. Tests live in `__tests__/` (unit) or `tests/` (integration). Mirror the source path - `lib/enhance/fixer.js` tests in `__tests__/enhance-fixer.test.js` or `tests/enhance/fixer.test.js`.

## Architecture: Plugins, Commands, Agents, Skills

**19 plugins** - standalone repos under the `agent-sh` org, installed via `bin/cli.js`.

**Discovery** is automatic via `lib/discovery/`:
- `plugins/<name>/commands/*.md` → Commands
- `plugins/<name>/agents/*.md` → Agents
- `plugins/<name>/skills/*/SKILL.md` → Skills

**Command file** (`plugins/{name}/commands/{cmd}.md`) frontmatter:
```yaml
---
description: Short description for implicit invocation (max 500 chars)
codex-description: 'Use when user asks to "phrase1", "phrase2". What it does.'
---
```

**Codex trigger phrases are required** - without them Codex won't invoke the skill.

**Cross-platform env vars** - always use these patterns:
```javascript
// [OK] Works on all platforms
const pluginRoot = process.env.PLUGIN_ROOT || process.env.CLAUDE_PLUGIN_ROOT;
const stateDir = process.env.AI_STATE_DIR || '.claude';

// [ERROR] Only works on Claude Code
const pluginRoot = process.env.CLAUDE_PLUGIN_ROOT;
```

**OpenCode constraints:**
- All `AskUserQuestion` labels must be ≤30 characters
- Use `${PLUGIN_ROOT}` not `${CLAUDE_PLUGIN_ROOT}` in command/agent files
- Agent frontmatter is auto-transformed by installer (tools → permissions, model names)

**Windows-safe path requires:**
```javascript
// [OK]
const module = require('${CLAUDE_PLUGIN_ROOT}'.replace(/\\/g, '/') + '/lib/module.js');
// [ERROR]
const module = require('${CLAUDE_PLUGIN_ROOT}/lib/module.js');
```

## Agent Design

File location: `plugins/next-task/agents/{agent-name}.md`

Agent template structure:
```markdown
# Agent: {name}
## Role
{one-sentence description}
## Instructions
1. ALWAYS {critical constraint}
2. NEVER {prohibited action}
## Tools Available
- tool_1: description
If tool not listed, respond: "Tool not available"
## Output Format
<output>{exact structure}</output>
## Critical Constraints
{repeat most important constraints}
```

- Put critical info at START and END (Lost in Middle mitigation)
- Use explicit tool allowlisting
- Use imperative language ("Do X", not "You should do X")

Model assignment:
| Complexity | Model | Use For |
|------------|-------|---------|
| Complex reasoning | `opus` | exploration, planning, implementation, review |
| Standard tasks | `sonnet` | validation, cleanup, monitoring |
| Simple operations | `haiku` | worktree setup, simple fixes |

## Website (site/)

The website at `site/index.html` is **hardcoded HTML** - not generated from content.json or any template. When plugins, commands, agents, or skills are added/removed/renamed, the site must be manually updated:

- `site/index.html` - all counts (meta tags, hero stats, section headers), command tabs + panels, agent tier cards, skills grid
- `site/content.json` - commands array, stats, meta description, research section, recent_releases
- `site/ux-spec.md` - design spec counts

**Checklist when adding a new command:**
1. Add tab button to the commands tablist (with correct index)
2. Add tab panel with tagline, 4 features, code block (copy the SVG from an existing panel)
3. Update "N Commands. One Toolkit." heading count
4. Update all meta tag counts (description, og:description, twitter:description)
5. Update hero badge counts
6. Update stats bar data-target attributes
7. Add entry to content.json commands array
8. If new agent: add to agent tier cards + update tier counts
9. If new skill: add to skills grid + update skills count

**How It Works sections** for new commands must also be created manually in the HTML.

Current counts (v5.8.1): 19 plugins, 20 commands, 47 agents, 40 skills, 5 platforms.

## CI/CD

`.github/workflows/ci.yml` runs on push/PR to main:
1. `npm ci`
2. `npm test`
3. `npm run validate`
4. `node scripts/expand-templates.js --check`
5. `node scripts/gen-adapters.js --check`
6. `agnix` workflow (agent config linting)

Releases are triggered by `git tag vX.Y.Z` and publish to npm automatically.

## Version & Release

`package.json` is the single source of truth for version.

```bash
npx agentsys-dev bump X.Y.Z     # Stamps version into package.json, package-lock.json,
                                  # plugin.json, marketplace.json, site/content.json
npm run validate:consistency     # Verify all files agree
npm test                         # All tests pass
npm pack --dry-run               # Package builds correctly
git add -A && git commit -m "chore: release vX.Y.Z"
git tag vX.Y.Z
git push origin main --tags      # Triggers release.yml → publishes to npm
```

Pre-release tags: `vX.Y.Z-rc.1` (npm tag: `rc`), `vX.Y.Z-beta.1` (npm tag: `beta`).

Plugin repos have independent versions - not stamped by the bump command.

## Checklists

Detailed step-by-step guides in `checklists/`:
- `new-command.md` - Adding a slash command
- `new-agent.md` - Adding a specialist agent
- `new-skill.md` - Adding a skill
- `new-lib-module.md` - Adding a lib module
- `release.md` - Full release process
- `cross-platform-compatibility.md` - Master cross-platform reference
- `update-opencode-plugin.md` - OpenCode plugin updates

## References

- Part of the [agentsys](https://github.com/agent-sh/agentsys) ecosystem
- https://agentskills.io
- `docs/ARCHITECTURE.md` - System architecture deep-dive
- `docs/CROSS_PLATFORM.md` - Multi-platform implementation details
- `docs/TESTING.md` - Test strategy and conventions
