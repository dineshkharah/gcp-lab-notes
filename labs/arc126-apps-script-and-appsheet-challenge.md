# ARC126, Develop with Apps Script and AppSheet challenge lab

Four checkpoints. Thirty minutes stated. No Cloud Shell and barely any cloud console: AppSheet, the Apps Script editor, the OAuth consent screen, and Google Chat.

This is the challenge lab at the end of the Develop with Apps Script and AppSheet badge, and it is almost entirely a replay of three guided labs. Nothing new is asked.

## It is GSP1146 plus GSP250, with different names

| ARC126 task | Comes from | Difference |
| --- | --- | --- |
| 1, create and customize an AppSheet app | `gsp1146-appsheet-no-code-chat-apps.md` | same app name `ATM Maintenance Tracker`, greeting says **would you like** rather than **do you want** |
| 2, add an automation | `gsp1146-appsheet-no-code-chat-apps.md` | identical, event `New ticket`, Adds only, `Tickets` table, message `You have created a new ticket.` |
| 3, create and publish an Apps Script bot | `gsp250-google-chat-bots-apps-script.md` | project is **`Helper Bot`** not `Friendly Bot`, description **`Helper chat bot`** not `Apps Script lab bot` |
| 4, configure the OAuth consent screen | `gsp250-google-chat-bots-apps-script.md` | same, app name `Helper Bot` |

`gsp235-apps-script-sheets-maps-gmail.md` is the third lab in the badge and contributes the Apps Script editor mechanics: naming and saving before running, picking the right function in the dropdown, and Executions as the place to diagnose a failed run.

Read those three files first. The only real work here is not copying the wrong literal across.

## The literals that changed, and are easy to get wrong

- Greeting text: `Welcome to the ATM Maintenance Tracker app. What would you like to do today?` **Would you like**, where GSP1146 uses **do you want**. Same sentence otherwise.
- Apps Script project name and Chat app name: `Helper Bot`
- Description: `Helper chat bot`
- Avatar URL is unchanged: `https://goo.gl/kv2ENA`

Everything unspecified stays at its default, which the task text says explicitly for tasks 1 and 2.

## The spreadsheet shortcut

`assets/ARC126.xlsx` is the Tickets sheet used for this run, the same technique as GSP1146: copy the template app, open its backing spreadsheet in Google Sheets, select the **Tickets** tab, then File, Import, **Replace current sheet**.

On GSP1146 that alone carried all three of its checkpoints without any Chat app builder work. Worth trying the same here and clicking the checkpoints before building the automation and the bot by hand.

Two caveats. Tasks 3 and 4 are Apps Script and OAuth, so no spreadsheet import can satisfy them; those have to be done. And this is one run against a manual dated January 2026 whose last tested date is older still, August 2025, so the scorers may not behave the same way twice.

## Task 3 and 4, the order that matters

The OAuth consent screen checkpoint is listed **last** in the lab text but has to be configured **before** the bot can be published. Do it in this order:

1. Apps Script, **Chat app (Intermediate version)** starter, rename to `Helper Bot`
2. OAuth consent screen: Get Started, app name `Helper Bot`, support email and contact email both the student address, Audience **Internal**
3. Copy the **Project number** from Cloud Overview, Dashboard, and set it in Apps Script Project Settings, Change project
4. Deploy, Test Deployments, copy the **Head Deployment ID**
5. Google Chat API, Configuration tab, fill in the table above, Save, then set App Status to `LIVE - available to users` and Save **again**
6. Google Chat, Start a chat, search `Helper Bot`

`Internal` audience, project **number** not id, and the second save on App Status are the three things that cost time. All three are written up in `gsp250-google-chat-bots-apps-script.md`.

## Cleanup

Do not delete the AppSheet app until all four checkpoints are green. GSP1146 ends with a delete step and it removes what the AppSheet checkpoints inspect.
