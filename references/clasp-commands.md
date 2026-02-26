# clasp CLI — Complete Command Reference

All commands for `@google/clasp` v3+. Global flags apply to every command.

## Global Flags

| Flag | Description |
|------|-------------|
| `--user <name>` | Use named credentials (default: `"default"`) |
| `--adc` | Use Application Default Credentials from environment |
| `--project <file>` | Read settings from alternate `.clasp.json` |
| `--ignore <file>` | Use custom ignore patterns file instead of `.claspignore` |
| `--json` | Output as JSON (machine-readable) |
| `--port <num>` | Custom port for OAuth callback during `login` |

---

## Authentication

### `clasp login`
Authorize access to your Google account.
```bash
clasp login
clasp login --user work              # Named credential
clasp login --creds client_secret.json  # Custom OAuth (org GCP project)
clasp login --adc                    # Application Default Credentials
clasp login --port 8080              # Custom callback port
```
Stores credentials in `~/.clasprc.json` (global) and `.clasprc.json` (local).

### `clasp logout`
Remove stored credentials.
```bash
clasp logout
clasp logout --user work
```

### `clasp show-authorized-user`
Display current authorized account.
```bash
clasp show-authorized-user
clasp show-authorized-user --json
```

---

## Project Management

### `clasp create-script`
Create a new Google Apps Script project.
```bash
clasp create-script --title "My Script"
```
Creates `.clasp.json` and `appsscript.json` in the current directory.

### `clasp clone-script`
Download an existing project locally.
```bash
clasp clone-script <scriptId>
clasp clone-script <scriptId> --rootDir src/
# scriptId: from script.google.com → Project Settings → Script ID
# Also accepts the full editor URL
```

### `clasp delete-script`
Delete a project permanently.
```bash
clasp delete-script
clasp delete-script --force          # Skip confirmation
```

### `clasp list-scripts`
List all Apps Script projects you have access to.
```bash
clasp list-scripts
clasp list-scripts --json
```

### `clasp open-script`
Open the project in the Apps Script web editor.
```bash
clasp open-script
```

### `clasp show-file-status`
Preview what clasp would push or pull (dry run).
```bash
clasp show-file-status
clasp show-file-status --json
```

---

## File Sync

### `clasp push`
Upload local files to Google.
```bash
clasp push
clasp push --watch                   # Re-push on every file change
clasp push --force                   # Skip interactive confirmation
```
Respects `.claspignore`. Pushes all `.gs`, `.js`, `.ts`, `.html` files by default.

### `clasp pull`
Download current files from Google.
```bash
clasp pull
clasp pull --versionNumber 3         # Pull a specific version
```

---

## Versions

### `clasp create-deployment`
Create a new deployment (and optionally a new version snapshot).
```bash
clasp create-deployment --description "v1.0"
clasp create-deployment --versionNumber 3 --description "v1.1 from version 3"
```

### `clasp list-deployments`
List all deployments for the project.
```bash
clasp list-deployments
clasp list-deployments --json
```

### `clasp update-deployment`
Update an existing deployment to point to a different version.
```bash
clasp update-deployment <deploymentId> --versionNumber 4
clasp update-deployment <deploymentId> --description "hotfix"
```

### `clasp delete-deployment`
Remove one or all deployments.
```bash
clasp delete-deployment <deploymentId>
clasp delete-deployment --all
```

---

## Execution & Logging

### `clasp run-function`
Execute a function in the deployed script remotely.
```bash
clasp run-function myFunction
clasp run-function processData --params '["arg1", 42]'
```
**Requirement:** Script must be deployed as an API Executable, or have a linked GCP project with Apps Script API enabled. The function must be a top-level function (not inside a class or object).

### `clasp tail-logs`
Stream execution logs from Cloud Logging (Stackdriver).
```bash
clasp tail-logs
clasp tail-logs --watch              # Continuous streaming
clasp tail-logs --json               # JSON output
clasp tail-logs --simplified         # Condensed format
```
**Requirement:** `exceptionLogging: "STACKDRIVER"` in `appsscript.json` and a linked GCP project.

### `clasp setup-logs`
Configure Cloud Logging for the project.
```bash
clasp setup-logs
clasp setup-logs --json
```

---

## API Management

### `clasp enable-api`
Enable a Google API for the project.
```bash
clasp enable-api sheets
clasp enable-api drive
clasp enable-api gmail
```

### `clasp disable-api`
Disable a Google API.
```bash
clasp disable-api calendar
```

### `clasp list-apis`
List all available APIs and which are enabled.
```bash
clasp list-apis
clasp list-apis --json
```

---

## Settings

### `clasp setting`
Read or update project settings.
```bash
clasp setting                        # Show current settings
clasp setting scriptId               # Show specific setting
clasp setting scriptId <newId>       # Update setting
```

---

## MCP / AI Integration

### Claude Code plugin
```bash
claude mcp add clasp -- npx -y @google/clasp mcp
# OR
/plugin install @google/clasp
```

### Gemini CLI
```bash
gemini extensions install https://github.com/google/clasp
```

---

## Debug & Proxy

```bash
# Enable verbose debug logging for clasp itself
DEBUG=clasp:* clasp <command>

# Proxy settings (corporate networks)
export HTTP_PROXY=http://proxy.company.com:8080
export HTTPS_PROXY=https://proxy.company.com:8080
```

---

## Required OAuth Scopes

For full clasp functionality, these scopes must be authorized:
- `https://www.googleapis.com/auth/script.deployments`
- `https://www.googleapis.com/auth/script.projects`
- `https://www.googleapis.com/auth/drive`
- `https://www.googleapis.com/auth/logging`
- `https://www.googleapis.com/auth/servicemanagement`

---

## v2 → v3 Migration Reference

| Old (v2) | New (v3) |
|----------|----------|
| `clasp open` | `clasp open-script` |
| `clasp deploy` | `clasp create-deployment` |
| `clasp deploy -i <id>` | `clasp update-deployment <id>` |
| `clasp undeploy <id>` | `clasp delete-deployment <id>` |
| `clasp undeploy --all` | `clasp delete-deployment --all` |
| `clasp versions` | No direct equivalent; use `list-deployments` |
| `clasp version <desc>` | Part of `create-deployment` |
| `clasp apis enable <api>` | `clasp enable-api <api>` |
| `clasp apis disable <api>` | `clasp disable-api <api>` |
| `clasp apis list` | `clasp list-apis` |
| `clasp logs` | `clasp tail-logs` |
| `clasp logs --setup` | `clasp setup-logs` |
| `clasp run <fn>` | `clasp run-function <fn>` |
| `clasp status` | `clasp show-file-status` |
| `clasp list` | `clasp list-scripts` |
| `clasp create` | `clasp create-script` |
| `clasp clone` | `clasp clone-script` |
| `clasp delete` | `clasp delete-script` |

**Removed in v3:** TypeScript transpilation. Use Rollup or esbuild externally.
