# arc131, Speech to Text challenge lab

Transcribe two audio files with the Speech to Text api, one English and one Spanish. Everything runs in an ssh session on the provisioned vm, not Cloud Shell, because the scorer looks for the request and response files on that instance.

## Task 1, api key

Console only. APIs and Services, then Credentials, then Create credentials, then API key. The newer panel makes **Select API restrictions** required, so pick Cloud Speech to Text API. Leave "Authenticate API calls through a service account" unchecked, because the requests use a plain `?key=` parameter and binding it to a service account breaks that.

## Tasks 2 and 3, the requests

The file names are different per task and do not follow a pattern. One run used `speech_request_en.json` with `response.json`, then `request_sp.json` with `speech_response_sp.json`. Read them off the lab page.

English. The wav file is stereo, so `audioChannelCount` is required or the api returns `INVALID_ARGUMENT` about channel count.

```
{
  "config": {
    "encoding": "LINEAR16",
    "languageCode": "en-US",
    "audioChannelCount": 2
  },
  "audio": {
    "uri": "gs://spls/arc131/question_en.wav"
  }
}
```

Spanish. The flac is mono, so leave the channel count out or it errors.

```
{
  "config": {
    "encoding": "FLAC",
    "languageCode": "es-ES"
  },
  "audio": {
    "uri": "gs://spls/arc131/multi_es.flac"
  }
}
```

Call it with:

```
export API_KEY=YOUR_KEY

curl -s -X POST -H "Content-Type: application/json" --data-binary @REQUEST_FILE \
"https://speech.googleapis.com/v1/speech:recognize?key=$API_KEY" > RESPONSE_FILE

cat RESPONSE_FILE
```

Check the response holds a `transcript`. An error body still creates the file, and a file with an error in it fails the checkpoint even though the file exists.

English transcript came back as "hi how far is it to the next train station" at 0.99 confidence.
