# Claude Skills

A collection of reusable [Claude Skills](https://docs.anthropic.com/en/docs/build-with-claude/skills) for productivity, research, and development workflows.

## Skills

| Skill | Category | Description |
|-------|----------|-------------|
| [safe-commit](./safe-commit/) | Workflow | Security-checked git commits with change classification |
| [journal-capture](./journal-capture/) | Productivity | Structured capture of decisions, reflections, brainstorms, progress, and teardowns |
| [project-insights](./project-insights/) | Analysis | Deep project health analysis across 5 dimensions |
| [research-gap-finder](./research-gap-finder/) | Research | Discover confirmed-open research gaps from academic papers (requires [discovery-dashboard](https://github.com/niyaznurbhasha/discovery-dashboard)) |
| [daily-digest](./daily-digest/) | Productivity | Personalized daily briefing from project data (requires [discovery-dashboard](https://github.com/niyaznurbhasha/discovery-dashboard)) |

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

## Customization

Each skill's `SKILL.md` is designed to be forked and adapted:
- **journal-capture**: Edit the project registry table and journal path to match your setup
- **project-insights**: Add your own repo paths and strategy doc locations
- **safe-commit**: Adjust commit prefixes and security checklist for your stack

## Dependencies

Most skills are standalone (no external dependencies). Two require the [discovery-dashboard](https://github.com/niyaznurbhasha/discovery-dashboard):
- `research-gap-finder` — calls the research gap scanning pipeline
- `daily-digest` — reads dashboard data for the briefing

## License

MIT
