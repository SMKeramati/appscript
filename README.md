# appscript

An agent skill for developing Google Apps Script projects locally using the `clasp` CLI — no browser copy-paste.

## Install

```bash
npx skills add <owner>/appscript -g -a claude-code
```

Or install to all supported agents:

```bash
npx skills add <owner>/appscript --all
```

## What it does

Teaches your AI coding agent the full local GAS development lifecycle:

- **Setup** — install clasp, authenticate, create or clone a project
- **Edit** — work with `.gs` files locally in your editor
- **Push/Pull** — sync with Google using `clasp push` / `clasp pull`
- **Run & Debug** — execute functions remotely, stream logs
- **Deploy** — version snapshots, web apps, API executables, add-ons

No MCP server, no build step. Just a SKILL.md that loads on demand.

## Skill structure

```
appscript/
├── SKILL.md                      ← main skill (loads when triggered)
└── references/
    ├── clasp-commands.md         ← full CLI reference + v2→v3 migration
    ├── gas-reference.md          ← quotas, runtime, gotchas, service patterns
    └── config-files.md           ← .clasp.json / appsscript.json / .claspignore
```

Reference files load on demand — only when the agent needs the detail.

## Requirements

- Node.js 22+
- `npm install -g @google/clasp`
- Apps Script API enabled: https://script.google.com/home/usersettings

## License

MIT
