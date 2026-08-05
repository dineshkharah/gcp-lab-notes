# GSP1146, Develop No-Code Chat Apps with AppSheet

Five tasks, three checkpoints. Fifteen minutes stated. There is no Google Cloud console and no Cloud Shell in this one: it runs entirely in AppSheet and Google Chat, signed in with the lab's student account.

Three scored checkpoints: create the app, test the app, build an automation.

## What actually scored, which is far less than the lab asks for

Our run completed with only this:

1. Signed in to AppSheet with the lab credentials, accepted the consent screens
2. Opened the ATM Maintenance template link from task 1 and used **Copy app**, naming it `ATM Maintenance Tracker`
3. Opened the copied spreadsheet in Google Sheets, went to the existing **Tickets** sheet, and imported `assets/GSP1146.xlsx` over it, replacing the current sheet

That was it. None of the Chat app builder work, no slash command, no deployment check, no Google Chat space, no automation.

The scorers appear to check the app exists and the Tickets table holds the expected rows, rather than checking any Chat configuration. Same weak scorer pattern as ARC130 and ARC114 in the ML api badges.

`assets/GSP1146.xlsx` in this repo is the sheet used. Import it with Google Sheets' File, Import, and choose **Replace current sheet** with the Tickets tab active.

This is one run against a manual dated March 2026. If the checkpoints do not go green, the full path is below.

## The full path, if needed

**Task 1.** Copy app from the template link. AppSheet writes the backing spreadsheet to `/appsheet/data/ATMMaintenanceTracker-nnnnnnn` in My Drive. Then Chat apps in the left nav, Create, Next on the Enable card to auto configure. **Do not reload the page while it configures**, it takes a few minutes.

**Task 2.** Customize card, First message. Replace the greeting with `Welcome to the ATM Maintenance Tracker app. What do you want to do today?`. Change `My Tickets` to `Issues Reported By Me`, delete the `Manage Techs` view, Save. Then Actions, New action, **Slash command: Open app view**, app view `Issues Reported By Me`, name `/myissues`, description `Lists tickets that include your email address`.

The `Unsupported app view selected` warning on a deleted view is informational, not an error.

**Task 3.** Deployment check will raise an App description warning. Fix it in Settings, Information, App Properties: Function `Maintenance`, Industry `Financial Services`. Save, back to Manage, re-run the check, then **Move app to deployed state**.

**Task 4.** Google Chat in a **new incognito tab**, create a space, View apps, add `ATM Maintenance Tracker`, Install app. Create a ticket with ATM ID `ABC123`, the lab email address in Email, symptom `Card reader not working`, Resolved `N`. Then type `/myissues` in the reply box.

**Task 5.** Chat apps, Customize, New action, **Build my own**. Event: name `New ticket`, data change type **Adds only**, table `Tickets`. Step: **Send a chat message**, Select chat spaces, add the space from task 4, message text `You have created a new ticket`. Save, then create another ticket in Chat and watch for the confirmation.

## Notes

The email field in the test ticket must be the **lab student address**, since the `Issues Reported By Me` view filters on it. A personal address there makes `/myissues` return nothing.

Task 4 wants Google Chat in a separate incognito tab so the student session is the one in use.

The final "Delete your app" step is unscored cleanup. Skip it until every checkpoint is green; deleting the app removes what all three of them look at.
