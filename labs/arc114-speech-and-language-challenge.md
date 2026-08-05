# ARC114, Analyze Speech and Language with Google APIs challenge lab

Four tasks, four checkpoints at 25 each. Ten minutes stated, and it is that short if you avoid the one trap below. Tasks 2 through 4 run on a provided vm called `lab-vm` over ssh; only the api key is made in Cloud Shell.

Manual last updated September 2023, which is much older than the rest of this badge family, and the scorers behave accordingly.

## The api key

Same shape as ARC130, but the key must cover **two** services because task 3 calls Speech:

```
gcloud services enable language.googleapis.com speech.googleapis.com apikeys.googleapis.com

gcloud alpha services api-keys create --display-name="arc114-key" \
  --api-target=service=language.googleapis.com \
  --api-target=service=speech.googleapis.com --quiet

export KEY_NAME=$(gcloud alpha services api-keys list \
  --filter="displayName=arc114-key" --format="value(name)")
export API_KEY=$(gcloud alpha services api-keys get-key-string $KEY_NAME --format="value(keyString)")
echo $API_KEY
```

A key restricted to language only will pass task 2 and fail task 3.

## The trap: an interactive `read` inside a pasted block

This is the one that cost the time, and it is a new failure mode worth naming.

The community script for this lab contains:

```
read -p "Enter your Google Cloud API Key: " API_KEY_INPUT
export API_KEY="$API_KEY_INPUT"
```

Paste that block into a shell and **`read` consumes the next line of the paste as its answer**. In our run `API_KEY` was set to the literal string `# Get API Key`, and the actual key, typed afterwards, was handed to bash as a command. The tell is visible in the terminal output:

```
Enter your Google Cloud API Key: # Get API Key
```

Both curls then sent a garbage key. Nothing errored loudly, because the script redirects into the response files, so `nl_response.json` and `speech_response.json` were written and contained a 403 `API_KEY_INVALID`.

The score pattern was the giveaway: tasks 1 and 4 full marks, tasks 2 and 3 both stuck at **10 of 25**. Those two are the only tasks that use the api key. Task 4 uses the Python client library and application default credentials, so it was unaffected.

Rule: **paste an `export VAR=...` line on its own, then check it.** A single `echo "[$API_KEY]"` with brackets around it would have caught this immediately, brackets included so an empty or whitespace value is visible.

Related and already in `gotchas.md`: a prompt inside a pasted block is how commands go missing. This is the mirror image, a pasted line going somewhere it was not meant to go.

## The recovery block

Once the key is set properly, only the two calls need redoing. Rewrite the request files anyway so nothing depends on the previous run:

```
export API_KEY=THE_LITERAL_KEY_STRING
echo "[$API_KEY]"

cat > nl_request.json <<'EOF'
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"With approximately 8.2 million people residing in Boston, the capital city of Massachusetts is one of the largest in the United States."
  },
  "encodingType":"UTF8"
}
EOF

cat > speech_request.json <<'EOF'
{
  "config": {
      "encoding":"FLAC",
      "languageCode": "en-US"
  },
  "audio": {
      "uri":"gs://cloud-samples-tests/speech/brooklyn.flac"
  }
}
EOF

curl -s -X POST -H "Content-Type: application/json" \
  --data-binary @nl_request.json \
  "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -o nl_response.json

curl -s -X POST -H "Content-Type: application/json" \
  --data-binary @speech_request.json \
  "https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" \
  -o speech_response.json

head -c 300 nl_response.json
cat speech_response.json
```

Expected: `entities` with Boston as `LOCATION`, and `"transcript": "how old is the Brooklyn Bridge"`.

Filenames are fixed by the lab: `nl_request.json`, `nl_response.json`, `speech_request.json`, `speech_response.json`. Note the underscores; ARC130 in the neighbouring badge uses hyphens for its equivalents.

The audio uri is given in the lab text and is a public bucket, `gs://cloud-samples-tests/speech/brooklyn.flac`, not one belonging to the project.

## Task 3 is Speech, task 2 is analyzeEntities

The lab says "pass your request body ... to the **Natural Language API**" in task 3, which is a copy paste error in the manual. Task 3 is `https://speech.googleapis.com/v1/speech:recognize`.

Task 2 is `analyzeEntities`, not `analyzeSentiment`, despite this badge being named for sentiment.

## Task 4 scored without the script running

Task 4 asks you to complete `analyze()` in a prewritten `sentiment_analysis.py`, download `gs://cloud-samples-tests/natural-language/sentiment-samples.tgz`, and run it on `reviews/bladerunner-pos.txt`.

In our run the script died with:

```
ModuleNotFoundError: No module named 'google'
```

The vm did not have `google-cloud-language` installed. The checkpoint still scored **25 of 25**. Same weak scorer behaviour as ARC130, where the Google Docs checkpoint scored with no document in existence.

So click task 4 before troubleshooting it. If it does need fixing, `pip3 install google-cloud-language` on the vm, then re-run. The working implementation, in the modern client style:

```python
def analyze(movie_review_filename):
    client = language_v1.LanguageServiceClient()

    with open(movie_review_filename) as review_file:
        content = review_file.read()

    document = language_v1.Document(
        content=content,
        type_=language_v1.Document.Type.PLAIN_TEXT
    )
    annotations = client.analyze_sentiment(request={"document": document})
    print_result(annotations)
```

Check which import style the provided file uses before writing this. The older samples use `from google.cloud import language` with `types.Document` and no `type_` keyword. Do not overwrite the provided file wholesale the way the community script does; it destroys `print_result` and the argument parsing that already work.

Unpacking the samples is two steps, and re-running it prompts:

```
gunzip sentiment-samples.tgz
tar -xvf sentiment-samples.tar
```

`gunzip` asks whether to overwrite if `sentiment-samples.tar` already exists, which is another interactive prompt to keep out of a pasted block.
