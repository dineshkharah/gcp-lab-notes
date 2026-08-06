# GSP038, Entity and Sentiment Analysis with the Natural Language API

Seven tasks, **three** checkpoints, fifteen minutes. Console for an api key, then everything else over ssh on a provisioned vm called `linux-instance`.

This is the parent lab for the Natural Language work in ARC130 and ARC114. Every request shape those challenge labs ask for comes from here.

## The three checkpoints are all in the first three tasks

- **Create an API Key** after task 1
- **Make an Entity Analysis Request** after task 2, which is just **writing `request.json`**, before any call happens
- **Check the Entity Analysis response** after task 3

Tasks 4 through 7, sentiment, entity sentiment, syntax and multilingual, are **unscored**. They are the interesting half of the lab but nothing depends on them.

## Api key, and the literal paste rule

Console: APIs & Services, Credentials, Create credentials, API key, restrict it to **Cloud Natural Language API**, Create, copy.

Then ssh to `linux-instance` and set it there:

```
export API_KEY=THE_LITERAL_KEY_STRING
echo "[$API_KEY]"
```

The key lives in the console, the shell lives on the vm, and nothing crosses between them. Paste the literal string, never `$API_KEY` from somewhere else. Same trap as GSP1317 and ARC114.

## Skip nano

The lab has you edit `request.json` five times with nano. A heredoc does the same thing and cannot be half saved:

```
cat > request.json <<'EOF'
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"Joanne Rowling, who writes under the pen names J. K. Rowling and Robert Galbraith, is a British novelist and screenwriter who wrote the Harry Potter fantasy series."
  },
  "encodingType":"UTF8"
}
EOF

curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json > result.json

cat result.json
```

Quoted `'EOF'` so nothing inside gets expanded by the shell.

## The four endpoints, which is what this lab is really for

All take the same `request.json` shape and differ only in the url:

| Endpoint | What it gives you |
| --- | --- |
| `analyzeEntities` | entities with `type`, `metadata.wikipedia_url`, `salience`, `mentions` |
| `analyzeSentiment` | `documentSentiment` plus a per sentence breakdown in `sentences[]` |
| `analyzeEntitySentiment` | sentiment attached to each **entity** rather than the document |
| `analyzeSyntax` | tokens with `partOfSpeech`, `dependencyEdge`, `lemma` |

`analyzeEntitySentiment` is the one with no equivalent in the challenge labs and the most useful idea in the lab. On `I liked the sushi but the service was terrible.` the document score is a useless average, while the entity breakdown gives `sushi` a 0 and `service` a -0.7. That is the difference between knowing a review was mixed and knowing what was wrong.

Two numbers on every sentiment object:

- `score`, -1.0 to 1.0, how positive or negative
- `magnitude`, 0 to infinity, how much sentiment is present regardless of direction

A long text full of strong opinions in both directions has a score near zero and a high magnitude. That pair is what the challenge labs never make you look at.

## A discrepancy in the lab text

Task 4 prints a response with both sentences at `score: 0.9`, and then the prose underneath says "the score for the first sentence is positive (0.7), whereas the score for the second sentence is neutral (0.1)". The prose does not match its own sample output.

Nothing is scored on it, and the lab warns that your numbers will differ anyway, so ignore the mismatch rather than trying to reproduce either set.

## Multilingual, task 7

```
cat > request.json <<'EOF'
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"日本のグーグルのオフィスは、東京の六本木ヒルズにあります"
  }
}
EOF

curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

No language is declared and no `encodingType` either; the api detects `ja` and returns Japanese wikipedia urls. ARC130's French Roppongi Hills request is the same exercise with a different language, and this is where it comes from.
