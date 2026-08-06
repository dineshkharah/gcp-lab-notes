# GSP126, Using the Natural Language API from Google Docs

Four tasks, **two** checkpoints, ten minutes. No Cloud Shell. The console for an api key, then Google Docs and Apps Script.

This is the source of ARC130's task 2. If you have done that challenge lab, this is the same script written out in stages instead of handed over whole.

## Both checkpoints land before the api is ever called

- **Get an API Key** after task 2
- **Set up your Google Doc** after task 3, which deploys the script with a **stub** `retrieveSentiment` that always returns `0.0`

Task 4, the part that actually calls the Natural Language API, is **unscored**. So the lab is complete at the point where every highlight comes out yellow.

Worth doing task 4 anyway, it is the whole point, but it is not what the score depends on.

## Task 2, the api key

Console only: APIs & Services, Credentials, Create credentials, API key. The lab has you restrict it to **Cloud Natural Language API** in the create panel before clicking Create.

Note the contrast within this badge. GSP097 uses a service account json and no api key at all; ARC130 creates the key from the cli and hits the org policy that forces `--api-target`. Here the console does the restricting for you, so nothing fights back.

Copy the key before closing the dialog.

## Task 3, and the three things that make it look broken

Extensions, Apps Script from inside the doc. Delete the stub `myFunction`, paste the lab's code, save.

1. **Reload the document** after saving. `onOpen` only fires on load, so the *Natural Language Tools* menu does not exist until you do. This is the most common reason people think the script did not save.
2. **Select text before choosing Mark Sentiment.** With nothing selected the function runs, finds no selection, and does nothing at all. No error.
3. **Authorization runs on first use**: OK, choose the account, allow the script to view and manage documents it is installed in. `@OnlyCurrentDoc` at the top of the file is what keeps that consent scoped to this one document.

Everything highlights **yellow** at this stage, because the stub returns `0.0` which sits between the two cutoffs. That is the expected result, not a failure.

Any text works. The lab suggests pasting Alice in Wonderland from Project Gutenberg, but a couple of sentences with obvious tone make the colours easier to read.

## Task 4, wiring up the real call

Only `retrieveSentiment` changes. Replace the stub body with the api key and the fetch:

```javascript
function retrieveSentiment (line) {
  var apiKey = "YOUR_API_KEY";
  var apiEndpoint = "https://language.googleapis.com/v1/documents:analyzeSentiment?key=" + apiKey;

  var docDetails = { language: 'en-us', type: 'PLAIN_TEXT', content: line };
  var nlData = { document: docDetails, encodingType: 'UTF8' };
  var nlOptions = { method: 'post', contentType: 'application/json', payload: JSON.stringify(nlData) };

  var response = UrlFetchApp.fetch(apiEndpoint, nlOptions);
  var data = JSON.parse(response);

  var sentiment = 0.0;
  if (data && data.documentSentiment && data.documentSentiment.score) {
    sentiment = data.documentSentiment.score;
  }
  return sentiment;
}
```

Save, **reload the document again**, and re-authorize if prompted; the new `UrlFetchApp` call needs an external request scope the first version did not.

Thresholds in `markSentiment` are `-0.2` and `+0.2`, so anything in between still comes out yellow. A genuinely mild sentence being yellow is correct behaviour.

## The bit worth actually trying

The lab's optional step is the interesting one. Analyse `I'm happy. I'm happy. I'm sad.` and then add another `I'm sad.`. `documentSentiment.score` is an average across the document, so the colour flips as the balance shifts, while each individual sentence keeps its own score in `sentences[]`. That is the difference between document level and sentence level sentiment, which the challenge lab never makes you look at.
