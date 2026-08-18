# GSP073, Cloud Storage: Qwik Start Google Cloud Console

Five tasks, **three** checkpoints, ten minutes stated and **no cost**. About **five minutes**. **Console only.**

Manual and last tested both January 2026.

The console twin of `gsp074-cloud-storage-qwik-start-cli-sdk.md`. Same product, same four objectives, and **a different correct answer for making an object public**, which is the one thing in here worth carrying away.

## Three checkpoints, and tasks 4 and 5 score nothing

| Checkpoint | What it reads |
|---|---|
| 1 | the bucket exists |
| 2 | `kitten.png` in the bucket |
| 3 | `kitten.png` publicly readable |

Tasks 4 and 5, the folders and the delete, are unscored.

## Task 5 deletes the bucket, not a folder

**The heading says Delete a folder and the steps delete the entire bucket**: return to the buckets level, select the bucket, click Delete, type `DELETE`. That takes out `kitten.png`, `folder1`, `folder2` and everything all three checkpoints were scored on.

So **claim all three checkpoints before task 5**. Eighth instance of this shape in these notes, and the first where the task title actively misdescribes what the task does.

Also in the same task group: task 4 says to upload *"the screenshot that you downloaded"* into `folder2`. **There is no screenshot anywhere in this lab.** It is a leftover from an earlier version of the manual. Nothing is scored on it, so any file will do, including `kitten.png` again.

## The bucket settings are not defaults, and one of them is the trap

The lab dictates four specific choices at create time:

| Field | Value |
|---|---|
| Location type | Region, with the region filled in at lab start |
| Default class | Standard |
| Access control | **Uniform** |
| Enforce public access prevention | **unchecked** |

**Unchecking public access prevention is the one that matters.** Leave it on and task 3's `allUsers` grant is refused, no matter how correctly it is entered, because that setting overrides IAM rather than being negotiated with it. The lab flags it twice, once in the field list and again as a note about the Public access will be prevented prompt, which is the tell that it catches people.

## Task 3 is a bucket level IAM binding, despite the checkpoint's name

```
New principals   allUsers
Role             Cloud Storage > Storage Object Viewer
```

Granted on the **bucket**, from the Permissions tab.

**The checkpoint is named "Share a kitten.png object publicly" and the action is not on the object at all.** It cannot be: Access control was set to Uniform at create time, and uniform bucket level access forbids per object ACLs outright. So the object becomes public by inheriting a bucket binding, and the checkpoint verifies the effect rather than the mechanism.

Worth stating plainly because the neighbouring labs are the exact inverse. Now a section in `gotchas.md`.

Verification the console offers: the object's **Public access** column reads `Access granted to public principals`, and **Copy URL** yields `https://storage.googleapis.com/BUCKET/kitten.png`. Refresh if the column looks stale, which the lab also warns about.

## The two quiz answers

| Question | Answer |
|---|---|
| Every bucket must have a unique name across the entire Cloud Storage namespace | **True** |
| Object names must be unique only within a given bucket | **True** |

Both true, and the pair is the point: bucket names are global, object names are scoped to their bucket.

## If it ever needs doing from the shell

Nothing here requires the console, and the CLI equivalent is short. Kept for reference rather than because it was used:

```
export PROJECT_ID=$(gcloud config get-value project)
gcloud storage buckets create gs://$PROJECT_ID --location=REGION --default-storage-class=STANDARD --uniform-bucket-level-access --no-public-access-prevention
gcloud storage cp kitten.png gs://$PROJECT_ID/
gcloud storage buckets add-iam-policy-binding gs://$PROJECT_ID --member=allUsers --role=roles/storage.objectViewer
curl -s -o /dev/null -w "public HTTP %{http_code}\n" https://storage.googleapis.com/$PROJECT_ID/kitten.png
```

Note `--no-public-access-prevention` and `add-iam-policy-binding` rather than `objects update --add-acl-grant`. With uniform access on, the ACL form is not available.

## Related files

- `gsp074-cloud-storage-qwik-start-cli-sdk.md`, the CLI twin, where the public grant is an **object ACL** because that bucket has fine grained access.
- `arc111-cloud-storage-data-protection-challenge.md`, where uniform access has to be **turned off** and only an object ACL passes.
- `gsp421-apis-explorer-cloud-storage.md` for the same operations through the JSON API form.
