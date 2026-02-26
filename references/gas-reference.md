# Google Apps Script — Runtime, Quotas & Gotchas

Reference for AI agents writing or debugging GAS code.

---

## Runtime: V8 vs Rhino

Always use V8 (`"runtimeVersion": "V8"` in `appsscript.json`) unless maintaining legacy code.

| Feature | V8 (Recommended) | Rhino (Legacy) |
|---------|-----------------|----------------|
| JS engine | Chrome V8 | Mozilla Rhino |
| ES support | ES6+ (arrow functions, classes, async/await, destructuring) | ES5 only |
| Performance | Fast | Slower |
| Private class fields (`#field`) | Not supported | Not supported |
| Static class field declarations | Not supported | Not supported |

---

## Execution Limits

| Limit | All Accounts |
|-------|-------------|
| Script execution time | **6 minutes** (hard limit) |
| Custom function (Sheets formula) | 30 seconds |
| Add-on runtime | 30 seconds |
| Simultaneous executions per user | 30 |
| Simultaneous executions per script | 1,000 |

**Workaround for long operations:** Use `ContinuationTokens`, split into batches, or chain triggers.

---

## Daily Quotas

### Email (GmailApp)
| Operation | Consumer | Workspace |
|-----------|----------|-----------|
| Recipients/day | 100 | 1,500 |
| Read/write operations | 20,000 | 50,000 |

### URL Fetch (UrlFetchApp)
| Operation | Consumer | Workspace |
|-----------|----------|-----------|
| Calls/day | 20,000 | 100,000 |
| Response size per call | 50 MB | 50 MB |
| URL length | 2 KB | 2 KB |

### Spreadsheet/Document Creation
| Operation | Consumer | Workspace |
|-----------|----------|-----------|
| New Spreadsheets/day | 250 | 3,200 |
| New Documents/day | 250 | 1,500 |

Quotas reset 24 hours after first request. Quota errors: `"Service using too much computer time for one day"`.

---

## Language Restrictions

GAS is **not a browser** and **not Node.js**. These are unavailable:

```javascript
// ❌ NOT available
import { foo } from './utils.js'   // No ES modules
export default function() {}       // No exports
setTimeout(() => {}, 1000)         // No timers
setInterval(() => {}, 1000)
fetch('https://api.example.com')   // Use UrlFetchApp.fetch() instead
new Blob([data])                   // No Blob
window, document, localStorage     // No DOM/browser APIs
crypto.randomUUID()                // No Web Crypto
require('fs')                      // No Node.js built-ins
```

```javascript
// ✅ Correct alternatives
// HTTP requests:
const response = UrlFetchApp.fetch('https://api.example.com/data');
const json = JSON.parse(response.getContentText());

// Async is supported (Promises, async/await work in V8):
async function myFunc() {
  const result = await somePromise;
}
// But: no event loop. All I/O is blocking. No setTimeout callbacks ever fire.
```

---

## Global Scope & File Order

All `.gs` files share a single global scope. Load order can matter:

```json
// .clasp.json: control push order if needed
{
  "scriptId": "...",
  "filePushOrder": ["Config.gs", "Utils.gs", "Code.gs"]
}
```

Variables and functions defined in any file are accessible in all others — no imports needed.

---

## Performance Patterns

### Batch read/write (critical for Sheets)
```javascript
// ✅ Fast — one API call
const values = sheet.getRange('A1:B100').getValues();
sheet.getRange('C1:C100').setValues(result);
SpreadsheetApp.flush();  // Commit writes before reading

// ❌ Slow — 100 API calls in a loop
for (let i = 1; i <= 100; i++) {
  const val = sheet.getRange(i, 1).getValue();
  sheet.getRange(i, 3).setValue(val * 2);
}
```

### SpreadsheetApp.flush()
Writes are buffered. Call `flush()` before you need to read values you just wrote:
```javascript
range.setValues(data);
SpreadsheetApp.flush();    // Flush write cache
const readBack = range.getValues();  // Now reads the new data
```

### Minimize service calls
JavaScript operations run in milliseconds; service calls take 100–500ms each. Batch everything.

---

## CacheService

Temporary key-value store (max 6 hours). **Not guaranteed to persist** — always handle `null`:

```javascript
const cache = CacheService.getScriptCache();
let data = cache.get('mykey');
if (!data) {
  data = JSON.stringify(expensiveFetch());
  cache.put('mykey', data, 1500);   // 25 minutes in seconds
}
return JSON.parse(data);
```

Limits: max value 100 KB, max key 250 chars, max 1,000 items, max TTL 6 hours.

---

## PropertiesService

Persistent key-value storage (no expiry). Three scopes:
- `PropertiesService.getScriptProperties()` — shared across all users of the script
- `PropertiesService.getUserProperties()` — per-user, persistent
- `PropertiesService.getDocumentProperties()` — per-document (add-ons only)

```javascript
const props = PropertiesService.getScriptProperties();
props.setProperty('API_KEY', 'secret-value');
const key = props.getProperty('API_KEY');
```

Use this for config, API keys, and state that must survive across executions.

---

## Triggers

**Simple triggers** (automatic, no authorization required):
- `onOpen(e)` — spreadsheet/doc/form opens
- `onEdit(e)` — cell edited
- `onInstall(e)` — add-on installed
- `onSubmit(e)` — form submitted

**Installable triggers** (set up via ScriptApp or the editor):
```javascript
// Time-based
ScriptApp.newTrigger('myFunction').timeBased().everyHours(1).create();

// On spreadsheet edit (installable — can do more than simple onEdit)
ScriptApp.newTrigger('myFunction')
  .forSpreadsheet(SpreadsheetApp.getActive())
  .onEdit().create();
```

List and delete triggers:
```javascript
ScriptApp.getProjectTriggers().forEach(t => ScriptApp.deleteTrigger(t));
```

---

## Common Services Quick Reference

```javascript
// Spreadsheets
const ss = SpreadsheetApp.getActiveSpreadsheet();
const sheet = ss.getSheetByName('Sheet1');
const data = sheet.getDataRange().getValues();   // 2D array

// Docs
const doc = DocumentApp.getActiveDocument();
const body = doc.getBody();
body.appendParagraph('Hello World');

// Gmail
GmailApp.sendEmail('user@example.com', 'Subject', 'Body text');

// Drive
const files = DriveApp.getFilesByName('report.csv');
const file = files.next();

// HTTP
const res = UrlFetchApp.fetch('https://api.example.com', {
  method: 'post',
  contentType: 'application/json',
  payload: JSON.stringify({ key: 'value' }),
  headers: { Authorization: 'Bearer ' + token }
});

// Calendar
const cal = CalendarApp.getDefaultCalendar();
cal.createEvent('Meeting', new Date(), new Date(Date.now() + 3600000));

// Logging (shows in Execution log and Stackdriver)
Logger.log('value is %s', someVar);
console.log('also works in V8');
```

---

## Web App Pattern

```javascript
// Code.gs
function doGet(e) {
  const page = e.parameter.page || 'index';
  return HtmlService.createHtmlOutputFromFile(page)
    .setTitle('My App')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  // process data...
  return ContentService.createTextOutput(JSON.stringify({ status: 'ok' }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

Deploy as Web App → set "Execute as" (you or user) and "Who has access".

---

## Manifest Scope Hints

Common scopes to add to `oauthScopes` in `appsscript.json`:

```json
"oauthScopes": [
  "https://www.googleapis.com/auth/spreadsheets",
  "https://www.googleapis.com/auth/gmail.send",
  "https://www.googleapis.com/auth/drive",
  "https://www.googleapis.com/auth/calendar",
  "https://www.googleapis.com/auth/script.external_request",
  "https://www.googleapis.com/auth/userinfo.email"
]
```

Use the minimum scopes needed. Undeclared scopes are auto-detected but it's best practice to declare them.

---

## Common Runtime Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Exception: Service invoked too many times` | Hit daily quota | Batch operations; add delays; use Workspace account |
| `Exceeded maximum execution time` | 6-minute limit hit | Split into smaller steps; chain via triggers |
| `ReferenceError: X is not defined` | File order issue or typo | Check `filePushOrder` in `.clasp.json` |
| `You do not have permission to...` | Missing OAuth scope | Add scope to `oauthScopes` in manifest and re-authorize |
| `Cannot read properties of null` | CacheService returned null | Always null-check cache reads |
| `HttpResponseException: Response Code: 429` | UrlFetch rate limit | Add `Utilities.sleep(1000)` between calls |
