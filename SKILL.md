---
name: appscript
description: Use this skill whenever the user wants to develop, edit, push, deploy, debug, or manage Google Apps Script (GAS) projects using the clasp CLI. Triggers on any mention of clasp, Apps Script, .gs files, script.google.com, appsscript.json, Google Sheets/Docs/Forms/Slides/Drive automation, GAS triggers, or local GAS development. Use this when someone wants to automate Google Workspace, build a Sheet macro, create a web app on GAS, set up a time trigger, or write any script that runs in Google's cloud — even if they don't say "clasp" explicitly.
license: MIT
compatibility: Requires Node.js 22+ and clasp CLI (npm install -g @google/clasp). Works with Claude Code, Cursor, Codex CLI, and any agent with shell access.
allowed-tools: Bash Read Write Edit Glob
---

# appscript — Google Apps Script via clasp

Develop, edit, and deploy Google Apps Script projects from the terminal. Full lifecycle: create/clone → edit locally → push → test → deploy. No browser copy-paste.

## Quick Start

```bash
# Verify clasp is installed (needs Node 22+)
clasp --version
npm install -g @google/clasp   # install if missing

# One-time: enable Apps Script API in your Google account
# https://script.google.com/home/usersettings → toggle ON
# Without this, every clasp command fails with an authorization error.

clasp login                                    # authenticate (opens browser)
clasp create-script --title "My Script"        # start a new project
# — or —
clasp clone-script <scriptId>                  # clone existing project
# scriptId: script.google.com → gear icon → Script ID
```

Edit your `.gs` files locally, then `clasp push`. That's the core loop.

---

## Project Structure

```
my-project/
├── .clasp.json           ← project config (scriptId, rootDir)
├── .claspignore          ← gitignore-style exclusions from push
├── appsscript.json       ← manifest (must exist and be pushed)
├── Code.gs               ← main logic
├── Utils.gs              ← helpers
└── ui.html               ← HTML for web apps / sidebars
```

All `.gs` files share a **global scope** — there are no modules, no `import`/`export`. A function defined in `Utils.gs` is automatically available in `Code.gs`. File load order matters for top-level declarations; control it with `filePushOrder` in `.clasp.json` if needed.

**Always use V8 runtime** in `appsscript.json`. V8 unlocks ES6+ (arrow functions, async/await, destructuring). Without it you're on ES5 Rhino.

```json
{
  "timeZone": "America/New_York",
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8"
}
```

---

## Daily Workflow

```bash
clasp show-file-status          # preview what will push/pull (dry run)
clasp push                      # upload local → Google
clasp push --watch              # auto-push on every file save
clasp push --force              # skip confirmation prompts

clasp pull                      # download Google → local
clasp pull --versionNumber 3    # pull a specific version
```

---

## Running & Debugging

```bash
# Execute a function remotely
clasp run-function myFunction
clasp run-function processData --params '["arg1", 42]'

# Stream Cloud Logging
clasp tail-logs
clasp tail-logs --watch         # continuous
clasp tail-logs --json          # machine-readable

# Debug clasp itself
DEBUG=clasp:* clasp push
```

`clasp run-function` requires the project to be deployed as an **API Executable** or have a GCP project linked with Apps Script API enabled. Without that setup, push and test directly via the Apps Script editor at script.google.com, or trigger from a linked Sheet.

---

## Versions & Deployments

Versions are immutable snapshots. Deployments point to a version and expose the script externally.

```bash
clasp create-deployment --description "v1.0 initial release"
clasp create-deployment --versionNumber 3 --description "hotfix"

clasp list-deployments
clasp update-deployment <deploymentId> --versionNumber 4 --description "v1.1"
clasp delete-deployment <deploymentId>
clasp delete-deployment --all
```

**Deployment types** (configured in `appsscript.json`):
- **Web App** — `doGet(e)` / `doPost(e)` required; accessible via a public URL
- **API Executable** — enables `clasp run-function` from external callers
- **Add-on** — distributable via Google Workspace Marketplace

---

## Authentication

```bash
clasp login                              # standard OAuth (opens browser)
clasp show-authorized-user               # confirm active account
clasp logout

# Multiple accounts
clasp login --user work
clasp login --user personal
clasp push --user work                   # specify per command

# CI/CD (no browser)
clasp login --adc                        # uses GOOGLE_APPLICATION_CREDENTIALS
clasp login --creds client_secret.json   # org GCP project OAuth
```

---

## API Management

```bash
clasp enable-api sheets                  # enable Sheets API for this project
clasp list-apis                          # see available and enabled APIs
clasp disable-api drive
```

---

## GAS Code Quality

### Batch all Sheets operations

Every call to a Google service (Sheets, Drive, Gmail, etc.) is an HTTP round-trip costing ~100–500ms. In a loop, 100 individual cell reads become 100 serial requests — often enough to hit the 6-minute execution timeout. Read and write in bulk instead.

❌ **100 API calls — slow, risks timeout:**
```javascript
for (let i = 1; i <= 100; i++) {
  const val = sheet.getRange(i, 1).getValue();
  sheet.getRange(i, 2).setValue(val * 2);
}
```

✅ **2 API calls — fast:**
```javascript
const vals = sheet.getRange('A1:A100').getValues();   // one read
const result = vals.map(([v]) => [v * 2]);
sheet.getRange('B1:B100').setValues(result);           // one write
SpreadsheetApp.flush();   // commit buffered writes before any subsequent read
```

`SpreadsheetApp.flush()` is needed because GAS buffers writes. Skipping it means a read immediately after a write may return stale data.

### Language limits

```javascript
// ✅ Works in V8
const fn = (x) => x * 2;
async function fetchData() { const res = await somePromise; }
const [a, ...rest] = array;

// ❌ Never works in GAS (any runtime)
import { foo } from './utils.js'   // no ES modules — global scope only
export default function() {}
fetch('https://api.example.com')   // use UrlFetchApp.fetch() instead
setTimeout(() => {}, 1000)         // no timer callbacks
window / document / localStorage   // no DOM or browser APIs

// ❌ V8 parse error — avoid
class Foo { #privateField = 1; }   // private class fields not supported
```

### Store config in PropertiesService

Hardcoding API keys or settings in `.gs` files means they're pushed to Google and visible to script editors. Store them in PropertiesService instead — it persists across deployments and stays out of source files.

```javascript
// Write once (e.g., a setup function)
PropertiesService.getScriptProperties().setProperty('API_KEY', 'your-secret');

// Read anywhere
const key = PropertiesService.getScriptProperties().getProperty('API_KEY');
```

---

## Common Patterns

**Web app (HTTP handler):**
```javascript
function doGet(e) {
  return HtmlService.createHtmlOutputFromFile('ui').setTitle('My App');
}
function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  // process...
  return ContentService.createTextOutput(JSON.stringify({ ok: true }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**Time-based trigger:**
```javascript
function setupTrigger() {
  ScriptApp.newTrigger('myFunction').timeBased().everyHours(1).create();
}
// To remove all triggers:
ScriptApp.getProjectTriggers().forEach(t => ScriptApp.deleteTrigger(t));
```

**External HTTP request:**
```javascript
const res = UrlFetchApp.fetch('https://api.example.com/data', {
  method: 'post',
  contentType: 'application/json',
  payload: JSON.stringify({ key: 'value' }),
  headers: { Authorization: 'Bearer ' + token }
});
const json = JSON.parse(res.getContentText());
```

**Cache expensive fetches (up to 6 hours):**
```javascript
const cache = CacheService.getScriptCache();
let data = cache.get('mykey');
if (!data) {
  data = JSON.stringify(UrlFetchApp.fetch('https://api.example.com').getContentText());
  cache.put('mykey', data, 1500);   // 25 minutes
}
return JSON.parse(data);
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Apps Script API has not been used` | API not enabled in account | https://script.google.com/home/usersettings → enable |
| `Could not find .clasp.json` | Not in project root | `cd` to project directory |
| `Authorization required` | OAuth scopes changed in manifest | Re-push, then re-authorize in the script editor |
| `Script timeout` | Exceeded 6-min execution limit | Split into smaller batches; chain via triggers |
| `Service using too much computer time` | Daily quota hit | Use batch operations; wait 24h; Workspace accounts have higher limits |
| Stale data after write | Writes are buffered | Call `SpreadsheetApp.flush()` before reading |
| Push confirmation loop | Interactive terminal prompt | Add `--force` flag |
| Command not found after upgrade | Clasp v2 → v3 renames | See rename table in `references/clasp-commands.md` |

---

## Reference Files

Load these on demand — don't pre-read them unless you need the detail.

| File | Load when |
|------|-----------|
| [`references/clasp-commands.md`](references/clasp-commands.md) | Need full flag reference for any command, or migrating from clasp v2 (full rename table there) |
| [`references/gas-reference.md`](references/gas-reference.md) | Need daily quotas, all available services, deployment manifest fields, more code patterns |
| [`references/config-files.md`](references/config-files.md) | Need complete `.clasp.json` / `appsscript.json` / `.claspignore` field reference |
