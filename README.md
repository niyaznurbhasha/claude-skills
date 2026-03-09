# Claude Skills

A collection of reusable [Claude Skills](https://docs.anthropic.com/en/docs/build-with-claude/skills) for productivity, research, and development workflows.

## Skills

### Standalone (works anywhere)

| Skill | Category | Description |
|-------|----------|-------------|
| [brain-dump](./brain-dump/) | Intake | Dump everything on your mind — auto-organizes into structured entries across projects with a review step before writing |
| [journal-review](./journal-review/) | Refinement | Review and correct categorizations — move items between projects, fix types, mark complete, create new projects |
| [journal-capture](./journal-capture/) | Capture | Structured capture of decisions, reflections, brainstorms, progress, and teardowns |
| [project-insights](./project-insights/) | Analysis | Deep project health analysis across 5 dimensions |
| [safe-commit](./safe-commit/) | Workflow | Security-checked git commits with change classification |

### Requires [discovery-dashboard](https://github.com/niyaznurbhasha/discovery-dashboard)

| Skill | Category | Description |
|-------|----------|-------------|
| [research-gap-finder](./research-gap-finder/) | Research | Discover confirmed-open research gaps from academic papers via arxiv + Semantic Scholar |
| [daily-digest](./daily-digest/) | Productivity | Personalized daily briefing from dashboard data |

## Installation

### Claude.ai
1. Download/clone this repo
2. Zip the skill folder you want (e.g., `safe-commit/`)
3. Go to Claude.ai → Settings → Capabilities → Skills
4. Upload the zip

### Claude Code
Place skill folders in your project directory or a shared skills directory. Claude Code will discover them automatically.

### API
Use the `/v1/skills` endpoint. See [Skills API docs](https://docs.anthropic.com/en/docs/build-with-claude/skills).

## The Brain Dump Workflow

The flagship workflow: dump everything on your mind in one shot, and it gets organized.

1. **brain-dump** — You speak/type a stream of consciousness. The skill extracts individual items, classifies each by type and project, groups related items, and shows you a proposal table.
2. **You review** — Confirm, correct any miscategorizations, approve new projects.
3. **journal-review** — If you need to fix things after writing, this skill handles moves, type corrections, completions, and project creation.

The goal: accurate categorization on the first pass so the review step is just a quick scan.

## Customization

Each skill's `SKILL.md` is designed to be forked and adapted:
- **brain-dump**: Add your project registry so it knows where to assign items
- **journal-capture**: Edit the project table and journal path to match your setup
- **project-insights**: Add your own repo paths and strategy doc locations
- **safe-commit**: Adjust commit prefixes and security checklist for your stack

## Dependencies

Most skills are standalone (no external dependencies). Two require the [discovery-dashboard](https://github.com/niyaznurbhasha/discovery-dashboard):
- `research-gap-finder` — calls the research gap scanning pipeline
- `daily-digest` — reads dashboard data for the briefing

## License

MIT
