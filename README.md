# zeroeval-skills

Agent skills for integrating ZeroEval into AI applications. Works with Cursor, Claude Code, Codex, and 30+ other coding agents.

## Skills included

**zeroeval-install** -- Get from zero to production-ready. Walks through SDK install (Python or TypeScript), first trace, `ze.prompt` migration, and starter judge recommendations.

**create-judge** -- Design and create automated judges. Covers binary vs scored evaluation, template writing, criteria design, and the full creation API.

**prompt-migration** -- Migrate hardcoded prompts to `ze.prompt`. Covers the full migration workflow, feedback wiring, judge linkage, staged rollout, and prompt optimization for both Python and TypeScript.

## Which skill do I need?

| I want to... | Use |
|-------------|-----|
| Add ZeroEval to my project for the first time | zeroeval-install |
| Set up tracing and observability | zeroeval-install |
| Migrate hardcoded prompts to ze.prompt | prompt-migration |
| Wire feedback collection for prompt optimization | prompt-migration |
| Connect judges to a prompt for automated evaluation | prompt-migration |
| Understand the staged rollout (explicit / auto / latest) | prompt-migration |
| Get recommendations for which judges to create | zeroeval-install |
| Create a new judge with a custom template | create-judge |
| Design scoring criteria for multi-dimensional evaluation | create-judge |
| Understand the judge creation API | create-judge |

## Install

### Option 1: CLI install (recommended)

```bash
# Install all skills
npx skills add zeroeval/zeroeval-skills

# Install a specific skill
npx skills add zeroeval/zeroeval-skills --skill zeroeval-install

# List available skills
npx skills add zeroeval/zeroeval-skills --list
```

### Option 2: Claude Code plugin

```bash
# Add the marketplace
/plugin marketplace add zeroeval/zeroeval-skills

# Install a specific plugin
/plugin install zeroeval-install@zeroeval-skills
/plugin install create-judge@zeroeval-skills
/plugin install prompt-migration@zeroeval-skills

# Reload plugins if the new commands do not appear immediately
/reload-plugins
```

Claude Code plugin skills are namespaced by plugin name. After installing from the marketplace, invoke them as:

```bash
/zeroeval-install:zeroeval-install
/create-judge:create-judge
/prompt-migration:prompt-migration
```

### Option 3: Clone and copy

```bash
git clone https://github.com/zeroeval/zeroeval-skills.git
mkdir -p .agents/skills
cp -r zeroeval-skills/skills/* .agents/skills/
```

Or copy individual skills:

```bash
# Cursor
mkdir -p .cursor/skills
cp -r zeroeval-skills/skills/zeroeval-install .cursor/skills/zeroeval-install
cp -r zeroeval-skills/skills/create-judge .cursor/skills/create-judge
cp -r zeroeval-skills/skills/prompt-migration .cursor/skills/prompt-migration

# Claude Code
mkdir -p .claude/skills
cp -r zeroeval-skills/skills/zeroeval-install .claude/skills/zeroeval-install
cp -r zeroeval-skills/skills/create-judge .claude/skills/create-judge
cp -r zeroeval-skills/skills/prompt-migration .claude/skills/prompt-migration
```

Note: on Windows without symlink support, use `npx skills` or the manual copy method. The `plugins/` directory contains symlinks that may not resolve on Windows.

## Repo structure

```
zeroeval-skills/
├── .claude-plugin/
│   └── marketplace.json               # Claude Code plugin marketplace
├── README.md
├── skills/                             # Canonical skill content (npx skills + manual copy)
│   ├── zeroeval-install/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── python-integration-playbook.md
│   │       ├── typescript-integration-playbook.md
│   │       ├── judges-playbook.md
│   │       └── troubleshooting.md
│   ├── create-judge/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── judge-creation-guide.md
│   │       ├── evaluation-patterns.md
│   │       └── defaults-and-api-reference.md
│   └── prompt-migration/
│       ├── SKILL.md
│       └── references/
│           ├── python-prompt-migration-playbook.md
│           └── typescript-prompt-migration-playbook.md
└── plugins/                            # Claude Code plugin wrappers (symlinked)
    ├── zeroeval-install/
    ├── create-judge/
    └── prompt-migration/
```

## Requirements

- A ZeroEval account and API key -- [zeroeval.com](https://zeroeval.com)
- Python 3.8+ or Node 18+
- An LLM provider SDK (OpenAI, Vercel AI, LangChain, etc.)
