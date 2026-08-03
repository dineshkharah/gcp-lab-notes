# How to run a lab

## Before starting

Collect the filled in values off the started lab page first: region, zone, project id, and every blank the task text refers to. The public overview page shows placeholders. The started lab page shows the real values.

Do not read the project id off a screenshot of the credentials panel. That field is clipped, and a project id that looks complete usually is missing its last characters. Use `$DEVSHELL_PROJECT_ID` in Cloud Shell instead.

Do not guess a blank from the pattern of a sibling task. One speech lab used `speech_request_en.json` and `response.json` for English but `request_sp.json` and `speech_response_sp.json` for Spanish. The scorer greps for the exact name.

## While working

- One task at a time. Run it, click Check my progress, confirm it went green, then move on.
- Paste ready command blocks beat clicking through the console. The console only wins when it hands you something for free, like the inline editor in the Cloud Run qwik start giving you a working function.
- Never put cleanup or delete steps in the same paste as the task they would erase. Confirm the checkpoint is green first, then delete. This cost a full rebuild on the Bigtable lab.
- End a block with a command whose output proves the scored thing exists. A create command that printed nothing is not proof that it ran.
- Long running work should be launched with `--async` where the flag exists, so the shell stays free.

## When a checkpoint will not go green

Work through these in order.

1. Read the scorer message, not just the score. It names the exact missing thing. This has found more failures than any amount of reasoning about config.
2. Check whether the command actually ran. In a long pasted block, individual commands go missing more often than you would expect. Search the terminal output for it.
3. Only after those two, start looking at the resource config.

## Cloud Shell notes

- Do not run `gcloud auth login`. Cloud Shell is already signed in and running it can leave the session with credentials that fail on every call.
- Do not run `gcloud config set project`. It is already correct. Pointing it at a wrong value produces `PERMISSION_DENIED`, `CONSUMER_INVALID`, and `RESOURCES_NOT_FOUND` errors that all look like missing permissions or a disabled api.
- Restarting Cloud Shell from the three dot menu clears bad session state and keeps the home directory.
- Open a second tab when a command blocks for minutes and you still need a shell.
- Env vars do not cross into a `docker run` container. Only what you pass with `-e` gets through, so set them again inside.
- `pip install` on the system python can be refused. Use a venv.
- A 429 on `serviceusage.googleapis.com` means the mutate quota for the minute is used up, usually by lab provisioning. Wait a minute and run it again.
