# ARC122, Analyze Images with the Cloud Vision API challenge lab

Three tasks, three checkpoints. Ten minutes stated and ten minutes real, which is rare. Nothing provisions and nothing propagates: an api key, two curl calls, two uploads.

Everything can be done from Cloud Shell including the api key, so there is no console work beyond clicking the checkpoints.

## What the lab actually wants

1. An api key in `$API_KEY`, and the bucket object made publicly readable
2. A `request.json` against the image with `TEXT_DETECTION`, response saved to `text-response.json` and uploaded to the bucket
3. The same with `LANDMARK_DETECTION` into `landmark-response.json`, also uploaded

The two detection checkpoints have the **same name**, "Analyze the image with the Cloud Vision API". Claim the text one before switching the json to landmark detection.

## The blanks in the task 2 template

The lab prints the json skeleton with the values missing, and it is also missing its commas:

```
"gcsImageUri":        ->  "gcsImageUri": "gs://BUCKET/IMAGE",
"type":               ->  "type": "TEXT_DETECTION",
```

Malformed json comes back as a 400 that reads like an authentication problem rather than a syntax problem.

## Discover the image name, do not guess it

The lab says only that a bucket "has been created for you with an image inside it". The citation at the bottom names *Manif des Sans-Papiers*, and the object did turn out to be `manif-des-sans-papiers.jpg` in our run, but that is a coincidence of the citation matching the filename, not a rule. Same class of mistake as the three bucket lab.

```
export IMG=$(gsutil ls "gs://$BUCKET/**" | grep -Ei '\.(jpg|jpeg|png)$' | head -1)
echo $IMG
```

That yields a full `gs://` uri ready to interpolate. Echo it before using it: an empty `$IMG` produces a request with an empty uri and an error that looks like an auth failure.

## The api key from the cli

No console needed:

```
gcloud services enable vision.googleapis.com translate.googleapis.com \
  language.googleapis.com apikeys.googleapis.com

gcloud alpha services api-keys create --display-name="vision-key" --quiet
export KEY_NAME=$(gcloud alpha services api-keys list --filter="displayName=vision-key" --format="value(name)")
export API_KEY=$(gcloud alpha services api-keys get-key-string $KEY_NAME --format="value(keyString)")
echo $API_KEY
```

`apikeys.googleapis.com` has to be enabled or the create fails. Two steps rather than reading `keyString` off the create call, because create returns a long running operation whose output shape varies by gcloud version.

Do **not** restrict the key to the Vision api. The lab's own standards say Vision, Translation and Natural Language should all be enabled, and a restricted key can only produce a 403 if a scorer probes something else. There is nothing to gain here.

## Public access, with a fallback

```
gsutil acl ch -u allUsers:R "$IMG" \
  || gcloud storage buckets add-iam-policy-binding gs://$BUCKET \
       --member=allUsers --role=roles/storage.objectViewer
```

Object acls do not work when the bucket has uniform bucket level access, and the failure is fatal in a script. The `||` falls through to a bucket level iam binding, which achieves the same thing.

## The whole lab runs on shell variables

`API_KEY`, `IMG` and `BUCKET` are plain shell variables. One unbroken Cloud Shell session start to finish, or they vanish and later commands fail with empty arguments.

## The community script for this lab

It works, and it did complete the lab in our run, but read the four problems before running it:

- **It ends with an interactive cleanup prompt** that deletes the api key and runs `gsutil -m rm -r gs://$PROJECT_ID-bucket/*`. That wipes both response files the checkpoints are scored on. A stray keystroke at that prompt is enough. Answer N deliberately, or delete the block.
- It hardcodes `manif-des-sans-papiers.jpg` and `exit 1`s if the object is not found under that name.
- It restricts the api key to Vision only, for no benefit.
- It `exit 1`s if `gsutil acl ch` fails, so a uniform access bucket stops it before any detection runs.

Minor: run as `./abhishek.sh` the `export API_KEY` dies with the subshell, so the key is not in your session afterwards. And its bonus `LABEL_DETECTION` step writes a third file that nothing scores.

## Verify before clicking

```
gsutil ls gs://$BUCKET
```

`text-response.json` and `landmark-response.json` both need to be there. Landmark detection on a photo of a street protest legitimately returns an empty `{}` response, which is not a failure: the checkpoint wants the file, not a landmark.
