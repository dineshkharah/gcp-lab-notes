# GSP097, Cloud Natural Language API: Qwik Start

Two tasks, two checkpoints, and genuinely about two minutes of work. Ten minutes stated.

Checkpoints: **Create an API Key** after task 1, **Make an Entity Analysis Request** after task 2.

## It does not actually use an api key

The checkpoint is called "Create an API Key" but nothing in this lab creates one. Task 1 makes a **service account** and downloads a **json key**:

```
export GOOGLE_CLOUD_PROJECT=$(gcloud config get-value core/project)

gcloud iam service-accounts create my-natlang-sa \
  --display-name "my natural language service account"

gcloud iam service-accounts keys create ~/key.json \
  --iam-account my-natlang-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com

export GOOGLE_APPLICATION_CREDENTIALS="$HOME/key.json"
```

Different mechanism from the rest of this badge family. ARC130 and ARC114 both create real api keys and pass `?key=` on a curl; this one uses application default credentials and the gcloud wrapper. Do not carry the `gcloud alpha services api-keys create` pattern over here, and do not expect the `--api-target` org policy problem from ARC130.

**The lab prints the last line as `export GOOGLE_APPLICATION_CREDENTIALS="/home/USER/key.json"` with a literal `USER`.** Use `$HOME/key.json` instead; pasting the placeholder is the same class of mistake as the `YOUR_DNS_RECORD` one in GSP1317.

## Task 2 runs on a vm, and does not need any of that

Compute Engine, VM instances, SSH on the provisioned instance. Then:

```
gcloud ml language analyze-entities --content="Michelangelo Caravaggio, Italian painter, is known for 'The Calling of Saint Matthew'." > result.json
cat result.json
```

Worth noticing: `GOOGLE_APPLICATION_CREDENTIALS` and `~/key.json` were set in **Cloud Shell**, not on the vm, so none of it reaches this command. It authenticates with the vm's own service account. Task 1's credentials exist to satisfy its checkpoint, not to enable task 2.

`gcloud ml language analyze-entities` is the gcloud wrapper over the same `documents:analyzeEntities` endpoint that ARC130 and ARC114 hit with curl. No request json, no api key.

## What the response teaches

Three entities from one sentence, and the fields are the point of the lab:

- `type`, so Caravaggio is `PERSON`, Italian is `LOCATION`, The Calling of Saint Matthew is `EVENT`
- `metadata.wikipedia_url` and `mid`, the knowledge graph link
- `salience`, 0 to 1, how central the entity is to the text. Caravaggio scores 0.83, the painting 0.03
- `mentions`, the same entity referred to more than once. Caravaggio appears as `PROPER` at offset 0 and as `COMMON` via the word "painter" at offset 33

The `language: en` at the bottom is auto detected, which is the same behaviour ARC130's French example relies on.
