# dev-conventions

Cross-project development and deployment conventions shared by **artbots**, **fedi-monitor**, **boekwinkeltjes-scraper**, and **fedi-dashboard**.

This is a documentation-only repository that defines how code is written, structured, configured, tested, and deployed across all projects in the bringolo ecosystem.

## Purpose

- **Consistency**: Anyone who has worked on one project can navigate any other without learning a new structure
- **Quality**: Enforced standards for cross-platform compatibility, error handling, logging, and testing
- **Onboarding**: Clear documentation for new developers (human or AI)
- **Compliance Tracking**: Audit matrix showing which projects follow which conventions

## Contents

| File | Purpose |
|------|---------|
| `development_principles.md` | 17 conventions covering how code is written, structured, and tested |
| `deployment_principles.md` | 22 conventions covering how code reaches the server |
| `development_audit.md` | Per-project compliance tracking with gap analysis |
| `claude_template.md` | Template for creating `CLAUDE.md` in new projects |
| `CLAUDE.md` | Guidance for Claude Code AI assistant when working in this repo |

## Projects Using These Conventions

| Project | Language | Description |
|---------|----------|-------------|
| [fedi-monitor](https://github.com/bringolo/fedi-monitor) | Python | Fediverse server health monitoring and discovery |
| [artbots](https://github.com/bringolo/artbots) | Node.js | Automated art posting bots for Mastodon and Bluesky |
| [boekwinkeltjes-scraper](https://github.com/bringolo/boekwinkeltjes-scraper) | Python | Dutch second-hand bookshop scraper with web dashboard |
| [fedi-dashboard](https://github.com/bringolo/fedi-dashboard) | Python | Web dashboard visualizing fedi-monitor data |

## Key Conventions

### Development

- **Cross-platform compatibility**: All code runs on Windows, Linux, and macOS
- **Configuration externalization**: No hardcoded values; config in JSON files, secrets in `.env`
- **File naming**: `lowercase_snake_case` everywhere (no camelCase, no kebab-case)
- **Database access**: Direct SQLite, no ORM
- **Logging**: Structured JSON logging with custom logger classes
- **Testing**: All tests must pass after every change

### Deployment

- **Directory structure**: `/srv/<project>/` with consistent subdirectories
- **Service users**: Dedicated system user per project with no login shell
- **Systemd services**: Hardened with `ProtectSystem`, `NoNewPrivileges`, etc.
- **Deploy scripts**: 8-phase pattern (stop, backup, pull, install, permissions, start, verify, report)
- **Port allocation**: 8005 (boekwinkeltjes), 8010 (artbots), 8015 (fedi-dashboard)

## Usage

### For New Projects

1. Copy `claude_template.md` to the new project as `CLAUDE.md`
2. Fill in the project-specific sections (tech stack, structure, architecture)
3. Follow the conventions in `development_principles.md` and `deployment_principles.md`
4. Add the new project to `development_audit.md` for compliance tracking

### For Existing Projects

- Reference these documents when making architectural decisions
- Check `development_audit.md` for known gaps to fix
- Update the audit when implementing new conventions

### For AI Assistants

Each project has its own `CLAUDE.md` that references these shared conventions. The `CLAUDE.md` file contains:
- A Code of Conduct for AI behavior (honesty, uncertainty, collaboration)
- Project-specific guidance
- References to the shared principles documents

## Contributing

When updating a convention:
1. Update the relevant `*_principles.md` file
2. Check if `development_audit.md` needs updating
3. Ensure all affected projects can comply with the change

## License

MIT
