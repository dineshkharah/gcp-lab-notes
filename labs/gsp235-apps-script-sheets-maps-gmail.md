# GSP235, Google Apps Script: Access Google Sheets, Maps & Gmail in 4 Lines of Code

Two checkpoints, ten minutes, and it really is ten minutes. No Cloud Shell, no cloud console at all. A Google Sheet, the bound Apps Script editor, and Gmail.

## The two checkpoints

1. **Create a new Google Sheet and enter a street address** after task 1
2. **Run the Google Sheets, Maps, and Gmail app** after task 4

Task 5 is a line by line explanation of the code and is unscored.

## Task 1

Open Google Sheets from the lab panel, signed in as the student account. Put a valid worldwide street address in **A1** and nothing else. `76 9th Ave, New York` works; the address needs a city or postal code or the Maps call has nothing to pin.

Claim the first checkpoint here.

## Tasks 2 and 3

**Extensions, Apps Script** from the Sheet's menu bar. That creates a **bound** script, one permanently tied to this spreadsheet, which is what makes `getActiveSheet()` work with no plumbing.

Replace the whole of `Code.gs`:

```javascript
/**
* @OnlyCurrentDoc
*/

function sendMap() {
    var sheet = SpreadsheetApp.getActiveSheet();
    var address = sheet.getRange("A1").getValue();
    var map = Maps.newStaticMap().addMarker(address);
    GmailApp.sendEmail("STUDENT_EMAIL_ADDRESS", "Map", 'See below.', {attachments:[map]});
}
```

Two things to get right:

- The email must be the **lab student address**, not a personal one. It is the inbox the lab has you check, and the checkpoint is about the mail actually going out.
- **Name and save the project before running.** An unsaved file shows a red dot next to the filename, and Apps Script will not let you proceed until the project has a name. Any name; `Hello Maps!` is the lab's suggestion.

`@OnlyCurrentDoc` is optional and scopes the authorization to this one spreadsheet rather than every sheet the user owns. Worth keeping, it narrows what the consent screen asks for.

## Task 4, the run and the authorization

The function dropdown still says `myFunction` after the paste. **Change it to `sendMap`** or Run does nothing useful.

First run triggers the authorization flow: Review Permissions, pick the student account, **Select all**, Continue. Apps Script writes the auth code itself; the consent is the human half of it.

Then check Gmail from the lab panel for a message with subject **Map** and the static map attached, and claim the second checkpoint.

If nothing arrives, **Executions** in the left rail of the Apps Script editor is the place to look. It lists each `sendMap` run with its status, and a failure there is more informative than an empty inbox.

## What the four lines touch

Three products in four lines, which is the whole point of the lab:

- `SpreadsheetApp.getActiveSheet()` then `getRange("A1").getValue()` reads the cell
- `Maps.newStaticMap().addMarker(address)` builds a static map with a pin
- `GmailApp.sendEmail(...)` sends it, with the map passed as `{attachments:[map]}`

Reading a different cell is a one word change, and `addMarker` can be called more than once for multiple addresses.
