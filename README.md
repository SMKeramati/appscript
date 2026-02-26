# appscript

An agent skill for developing Google Apps Script projects locally using the `clasp` CLI — no browser copy-paste.

## Prerequisites

Install clasp before using this skill:

```bash
npm install -g @google/clasp
clasp --version   # verify (needs Node.js 22+)
```

Also enable the Apps Script API once in your Google account:
https://script.google.com/home/usersettings → toggle **Google Apps Script API** ON

## Install

```bash
npx skills add SMKeramati/appscript -g
```

If Claude Code isn't picked up automatically:

```bash
npx skills add SMKeramati/appscript -g -a claude-code
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
- clasp: `npm install -g @google/clasp`
- Apps Script API enabled: https://script.google.com/home/usersettings

## License

MIT
