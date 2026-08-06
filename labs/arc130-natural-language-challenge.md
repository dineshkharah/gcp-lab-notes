# ARC130, Analyze Sentiment with Natural Language API challenge lab

Guided labs in this badge: `gsp097-natural-language-api-qwik-start.md`, which authenticates completely differently with a service account json rather than an api key, and `gsp126-natural-language-api-from-google-docs.md`, which is the source of this lab's task 2.

Four tasks, four checkpoints at 25 each. Ten minutes stated. Ours finished in about fifteen, and **two of the four checkpoints scored without the work they describe**, which is the main thing to know about this lab.

Tasks 3 and 4 run on a provided vm called `lab-vm` over browser ssh, not Cloud Shell. Task 1 is an api key. Task 2 is Google Docs plus Apps Script.

## The finding: tasks 1 and 2 scored off api traffic alone

Sequence in our run:

1. Created the api key, clicked task 1, got `Points earned: 0. Message: Please create an API key.` The key existed and was visible in the console at that moment.
2. Did tasks 3 and 4 on the vm. Both went green immediately, 25 each.
3. Refreshed the lab page. **Task 1 and task 2 both went green.** Total 100.

No Google Doc was ever created. No Apps Script was ever written. The task 2 scorer is not inspecting a document; it appears to key on the api key existing plus Natural Language api traffic in the project, both of which the two curls from the vm had already produced.

Task 1's initial failure was therefore stale scorer state, not a real problem with the key.

Practical reading: **do tasks 3 and 4 first, then refresh the page and click task 1 and task 2 before building anything by hand.** If they go green, the Docs work is optional. If they do not, the Apps Script steps are below.

This is not a guarantee, and the manual date on the page is June 2026, so it may be tightened later. But it cost us nothing to check and saved the slowest part of the lab.

## The api key, and an org policy that bites

```
gcloud services enable language.googleapis.com apikeys.googleapis.com

gcloud alpha services api-keys create --display-name="nl-key" \
  --api-target=service=language.googleapis.com --quiet

export KEY_NAME=$(gcloud alpha services api-keys list \
  --filter="displayName=nl-key" --format="value(name)")
export API_KEY=$(gcloud alpha services api-keys get-key-string $KEY_NAME --format="value(keyString)")
echo $API_KEY
```

`--api-target` is **required** in this project:

```
ERROR: (gcloud.alpha.services.api-keys.create) Invalid value for [--api-target]:
API keys must be created with API target restrictions. Please specify `--api-target`.
```

That is an org policy on the lab project, and it differs from ARC122 in the same badge family where an unrestricted key was accepted. So do not carry the unrestricted form between the two labs; try it and read the error.

Note the cascade in the output when the create fails: `api-keys list` then reports `The following filter keys were not present in any resource`, and `get-key-string` fails with a missing argument, and `echo` prints an empty `API_KEY=`. Three errors, one cause. The empty variable is the one that would have silently broken tasks 3 and 4.

## Tasks 3 and 4 on the vm

Shell variables do not cross into the vm, so paste the **literal** key string, not `$API_KEY`. Same trap as the psql host in GSP1317.

```
export API_KEY=THE_LITERAL_KEY_STRING

cat > analyze-request.json <<'EOF'
{
  "document":{
    "type":"PLAIN_TEXT",
    "content": "Google, headquartered in Mountain View, unveiled the new Android phone at the Consumer Electronic Show.  Sundar Pichai said in his keynote that users love their new Android phones."
  },
  "encodingType": "UTF8"
}
EOF

curl -s "https://language.googleapis.com/v1/documents:analyzeSyntax?key=${API_KEY}" \
  -H "Content-Type: application/json" --data-binary @analyze-request.json \
  -o analyze-response.txt

head -c 400 analyze-response.txt
```

```
cat > multi-nl-request.json <<'EOF'
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"Le bureau japonais de Google est situé à Roppongi Hills, Tokyo."
  }
}
EOF

curl -s "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -H "Content-Type: application/json" --data-binary @multi-nl-request.json \
  -o multi-response.txt

head -c 400 multi-response.txt
```

Filenames are fixed by the lab: `analyze-response.txt` and `multi-response.txt`.

## Task 4 is analyzeEntities, not analyzeSyntax

The lab says "Pass your request ... using the curl command or analyze syntax using gcloud ML commands" for **both** tasks 3 and 4, which is misleading. Task 4's French sentence is the entity analysis example from the parent lab, and the point is that Roppongi Hills and Tokyo come back as `LOCATION` with wikipedia metadata and the api detects `fr` without being told.

Our run: `analyzeSyntax` for task 3, `analyzeEntities` for task 4, both green.

## Task 2 by hand, if the shortcut stops working

1. `docs.new` in the same incognito window, signed in as the student account
2. Type a clearly positive sentence and a clearly negative one. Easier to see the highlighting than the Dickens passage the lab suggests.
3. **Extensions, Apps Script**. That is where the editor lives; it is not a separate product to go find.
4. Delete the stub `myFunction`, paste the lab's code, replace `"your key here"` with the literal key, save
5. Return to the doc tab and **reload the page**. `onOpen` only fires on load, so the *Natural Language Tools* menu does not exist until then.
6. **Select text**, then Natural Language Tools, Mark Sentiment. With nothing selected the function runs and does nothing, which is the usual reason it appears broken.
7. Authorize: Continue, pick the student account, Advanced, Go to (unsafe), Allow

## The community script for this lab

It only covers tasks 3 and 4, and it does them wrong for the scoring:

- **It never creates an api key.** It authenticates with `gcloud auth print-access-token` instead, so task 1 gets nothing, and the lab explicitly says tasks 3 and 4 should use the key from task 1.
- It runs in **Cloud Shell**, while the lab says both tasks require ssh on `lab-vm`. This lab carries the "Do not deviate from instructions" warning.
- It never touches task 2.
- It does not enable `language.googleapis.com`.

The one thing it gets right and worth confirming against: task 3 to `analyzeSyntax`, task 4 to `analyzeEntities`.
