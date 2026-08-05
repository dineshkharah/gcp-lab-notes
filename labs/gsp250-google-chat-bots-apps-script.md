# GSP250, Introduction to Google Chat Bots with Apps Script

Two tasks and **one** scored checkpoint. Ten minutes and it is genuinely that. No Cloud Shell, no gcloud. Apps Script, the OAuth consent screen, and the Google Chat API configuration page.

## The only checkpoint sits halfway through task 2

`Configure the user consent screen` is the single Check my progress in the whole lab. Everything after it, linking the Apps Script project to the cloud project, the head deployment id, the Chat API configuration, publishing, and the actual conversation with the bot, is **unscored**.

So the minimum for a green lab is:

1. Task 1: open the Apps Script link, pick the **Chat app (Intermediate version)** starter, rename the project to `Friendly Bot`
2. The OAuth consent screen configuration below
3. Click the checkpoint

The rest is worth doing to see the bot work, but nothing depends on it.

The challenge lab at the end of this badge, ARC126, replays this whole lab as `Helper Bot` and does score the publishing half. See `arc126-apps-script-and-appsheet-challenge.md`.

## Task 1

Sign in to the Google Cloud console **first**, then follow the Apps Script link from the lab. That order matters: the link inherits the signed in session, and hitting it first can land you on the wrong account.

Under **Google Workspace add-on starters** choose **Chat app (Intermediate version)**, then click `Untitled project` and rename to `Friendly Bot`.

`Code.gs` arrives prepopulated with the three event handlers, and reading them is the actual content of the lab:

- `onAddedToSpace()` for `ADDED_TO_SPACE`
- `onRemovedFromSpace()` for `REMOVED_FROM_SPACE`, which posts nothing back
- `onMessage()` for `MESSAGE`, a dm or an @mention in a room

No code is written in this lab.

## The OAuth consent screen, current flow

APIs & Services, OAuth consent screen, **Get Started**. The newer wizard splits this over four short steps:

| Step | Value |
| --- | --- |
| App Information, App name | `Friendly Bot` |
| User support email | the lab student address, from the dropdown |
| Audience | **Internal** |
| Contact Information | the same lab student address |

Accept the policy, Continue, Create. Then claim the checkpoint.

`Internal` is the one to get right. `External` sends you down a verification path the lab never asks for.

## The rest, unscored

**Link the Apps Script project to the cloud project.** Cloud Overview, Dashboard, copy the **Project number** (not the project id). In Apps Script, Project Settings gear, Google Cloud Platform (GCP) Project, Change project, paste the number, Set project.

**Get the head deployment id.** Apps Script, Deploy, Test Deployments, Copy next to **Head Deployment ID**. This is the test deployment, not a versioned one.

**Configure the Chat API.** APIs & Services, Library, Google Chat API (already enabled in the lab project), then its **Configuration** tab:

| Field | Value |
| --- | --- |
| App name | `Friendly Bot` |
| Avatar URL | `https://goo.gl/kv2ENA` |
| Description | `Apps Script lab bot` |
| Functionality | enable **Join spaces and group conversations** |
| Connection settings | **Apps Script**, paste the head deployment id |
| Visibility | the lab student address |

Save. Then scroll back to the top for **App Status**, set it to `LIVE - available to users`, and Save **again**. The App Status field often only appears after a page refresh, and the two saves are separate; one save is not enough.

**Test.** Google Chat, Start a chat, search `Friendly bot`, pick the one described as `Apps Script lab bot`. Adding it fires `onAddedToSpace` and you get a thank you message. Sending `Hello bot!` fires `onMessage` and it echoes `You said "Hello bot!"`.
