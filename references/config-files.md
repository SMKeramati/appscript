# clasp Config Files Reference

---

## `.clasp.json` — Project Configuration

Located in the project root (or wherever you run clasp commands from).

### Minimal (required)
```json
{
  "scriptId": "1BxSample1234567890AbCdEfGhIjKlMnOpQrStUvWxYz"
}
```

### Full options
```json
{
  "scriptId": "1BxSample1234567890AbCdEfGhIjKlMnOpQrStUvWxYz",
  "rootDir": "src/",
  "projectId": "my-gcp-project-id",
  "scriptExtensions": [".js", ".gs"],
  "htmlExtensions": [".html"],
  "filePushOrder": ["Config.gs", "Utils.gs", "Code.gs"],
  "skipSubdirectories": false
}
```

### Field descriptions

| Field | Default | Description |
|-------|---------|-------------|
| `scriptId` | — | **Required.** Apps Script project ID. Find at script.google.com → Project Settings → Script ID |
| `rootDir` | `.` | Local directory containing source files. Use `"src/"` to keep scripts in a subdirectory |
| `projectId` | — | GCP project ID. Required for Cloud Logging (`clasp tail-logs`), `clasp run-function`, and advanced API features |
| `scriptExtensions` | `[".js", ".gs"]` | File types treated as script code |
| `htmlExtensions` | `[".html"]` | File types treated as HTML templates |
| `filePushOrder` | — | Array of filenames pushed first (use for load-order dependencies) |
| `skipSubdirectories` | `false` | When true, skip files in subdirectories (backwards compat) |

**Finding your scriptId:**
1. Open your script at script.google.com
2. Click the gear icon (Project Settings)
3. Copy the "Script ID" field

**Or from a Google Sheets/Docs URL:**
In the Sheet URL `https://docs.google.com/spreadsheets/d/<SHEET_ID>/...`, the sheet ID ≠ script ID.
Go to Extensions → Apps Script to open the bound script and get its ID.

---

## `appsscript.json` — Project Manifest

The project manifest. **Must be present and pushed** with every deploy.
Created automatically by `clasp create-script` or `clasp clone-script`.

### Minimal
```json
{
  "timeZone": "America/New_York",
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8"
}
```

### Web App
```json
{
  "timeZone": "America/New_York",
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://www.googleapis.com/auth/spreadsheets",
    "https://www.googleapis.com/auth/userinfo.email"
  ],
  "webapp": {
    "executeAs": "USER_DEPLOYING",
    "access": "ANYONE"
  }
}
```

### API Executable
```json
{
  "timeZone": "America/New_York",
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "executionApi": {
    "access": "ANYONE"
  }
}
```

### With Advanced Services
```json
{
  "timeZone": "America/New_York",
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "dependencies": {
    "enabledAdvancedServices": [
      {
        "userSymbol": "Sheets",
        "serviceId": "sheets",
        "version": "v4"
      }
    ]
  }
}
```

### Field Reference

| Field | Values | Description |
|-------|--------|-------------|
| `timeZone` | ZoneId string | Project timezone, e.g. `"America/New_York"`, `"Europe/London"`, `"UTC"` |
| `runtimeVersion` | `"V8"`, `"STABLE"` | **Use V8.** STABLE = Rhino (legacy ES5) |
| `exceptionLogging` | `"STACKDRIVER"`, `"NONE"` | Where errors are logged. STACKDRIVER requires linked GCP project |
| `oauthScopes` | Array of scope URLs | Required Google API permissions. Declare only what you use |
| `webapp.executeAs` | `"USER_DEPLOYING"`, `"USER_ACCESSING"` | Run as script owner or as each user |
| `webapp.access` | `"MYSELF"`, `"DOMAIN"`, `"ANYONE"`, `"ANYONE_ANONYMOUS"` | Who can access the web app |
| `executionApi.access` | Same as webapp.access | Who can call API executable functions |
| `urlFetchWhitelist` | Array of HTTPS URL prefixes | Restrict which URLs UrlFetchApp can call |
| `dependencies.enabledAdvancedServices` | Array | Advanced Google APIs (Sheets v4, Drive v3, etc.) |

---

## `.claspignore` — Push Exclusions

Gitignore-style file controlling which local files clasp **will not push**.
Place in the project root (same level as `.clasp.json`).

**Default behavior (no .claspignore):** clasp pushes all `.gs`, `.js`, `.ts`, `.html` files recursively.

### Common patterns
```
# Dependencies
node_modules/
.git/

# Build artifacts
dist/
build/

# Test files
**/*.test.js
**/*.spec.js
tests/

# Documentation
*.md
docs/

# Config/secrets
.env
*.json
!appsscript.json   # ← Must re-include the manifest if you exclude *.json

# OS files
.DS_Store
Thumbs.db
```

**Critical:** If you add `*.json` to your ignore list, you must explicitly re-include `appsscript.json`:
```
*.json
!appsscript.json
```

### negation rules
- `!pattern` includes files that would otherwise be excluded
- Order matters: later rules override earlier ones
- Use `**` for recursive matching

---

## `~/.clasprc.json` — Global Credentials

Stored at `~/.clasprc.json` (home directory). Managed automatically by `clasp login`/`clasp logout`. Do not edit manually.

**Multiple accounts** are stored as named entries:
```json
{
  "default": { "token": {...} },
  "work": { "token": {...} },
  "personal": { "token": {...} }
}
```

Use `--user <name>` with any command to select an account.
