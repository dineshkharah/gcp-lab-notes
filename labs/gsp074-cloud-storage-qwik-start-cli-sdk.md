# GSP074, Cloud Storage: Qwik Start CLI/SDK

Eight tasks, **three** checkpoints, fifteen minutes stated and **no cost**. About **five minutes**. **Entirely Cloud Shell.**

Manual last updated May 2026, lab last tested February 2026.

## Three checkpoints out of eight tasks

| Checkpoint | What it reads |
|---|---|
| 1 | the bucket exists |
| 2 | `ada.jpg` inside `image-folder/` |
| 3 | `ada.jpg` at the bucket root is readable by `allUsers` |

Tasks 3, 5, 6 and 8 are unscored. They are there to demonstrate `cp` in the download direction, `ls`, `ls -l` and the removal of an ACL.

## The ordering trap

**Task 8 removes the public ACL and the step after it deletes `ada.jpg`, which is exactly what checkpoint 3 scores.** So checkpoint 3 has to be claimed before task 8 runs. Sixth instance of this shape in these notes and one of the clearer ones, since the destructive step is not even labelled as cleanup, it is the lab's next paragraph.

Checkpoint 2 is safe either way. The final `rm` targets `gs://BUCKET/ada.jpg` only, and the copy under `image-folder/` is a separate object that survives.

## The commands, in three blocks

Nothing here needs a value off the started page. The project id comes from the config and everything else is fixed in the lab.

**Block 1**, tasks 1 to 6, claims checkpoints 1 and 2:

```
export PROJECT_ID=$(gcloud config get-value project)
export BUCKET=$PROJECT_ID
gcloud storage buckets create gs://$BUCKET
curl -s https://upload.wikimedia.org/wikipedia/commons/thumb/a/a4/Ada_Lovelace_portrait.jpg/800px-Ada_Lovelace_portrait.jpg --output ada.jpg
ls -l ada.jpg
gcloud storage cp ada.jpg gs://$BUCKET
rm ada.jpg
gcloud storage cp gs://$BUCKET/ada.jpg .
gcloud storage cp gs://$BUCKET/ada.jpg gs://$BUCKET/image-folder/
gcloud storage ls gs://$BUCKET
gcloud storage ls -l gs://$BUCKET/ada.jpg
gcloud storage ls gs://$BUCKET/image-folder/
```

**Using the project id as the bucket name settles the global uniqueness rule for free.** Bucket names are unique across all of Cloud Storage, not per project, and a project id already satisfies every constraint the lab's naming list sets out: lowercase, three to sixty three characters, starts and ends alphanumeric, no `goog` prefix. The lab's own note about a `409 Bucket already exists` never comes up.

The last `ls` is the one that matters. It proves the object landed inside the folder rather than as an object literally named `image-folder/ada.jpg` at the root, which is the same thing in Cloud Storage but is what checkpoint 2 is looking at.

**Block 2**, task 7, claims checkpoint 3:

```
gcloud storage objects update gs://$BUCKET/ada.jpg --add-acl-grant=entity=allUsers,role=READER
curl -s -o /dev/null -w "public HTTP %{http_code}\n" https://storage.googleapis.com/$BUCKET/ada.jpg
```

`public HTTP 200` proves the ACL is live from outside the project, which is stronger than reading the acl back and much faster than the console round trip the lab asks for.

**Block 3**, task 8 and the delete, only after checkpoint 3 is green:

```
gcloud storage objects update gs://$BUCKET/ada.jpg --remove-acl-grant=allUsers
curl -s -o /dev/null -w "public HTTP %{http_code}\n" https://storage.googleapis.com/$BUCKET/ada.jpg
gcloud storage rm gs://$BUCKET/ada.jpg
gcloud storage ls -r gs://$BUCKET
```

`public HTTP 403` the second time, and the final listing leaves only `image-folder/ada.jpg`.

Note the flag asymmetry: `--add-acl-grant` takes `entity=allUsers,role=READER` while `--remove-acl-grant` takes the bare `allUsers`. The remove form identifies the entity, not the grant, so passing the full pair to it is an error.

## Setting the region does nothing to the bucket

The lab opens with `gcloud config set compute/region REGION` before task 1, which invites the assumption that the bucket lands there. It does not. That sets a **Compute** default, and `buckets create` with no `--location` gives the `US` multi region regardless, which is what the lab's own expected output shows.

So do not add a `--location` flag trying to match the region on the lab page. Related to the bucket location note in `gotchas.md`: location is fixed at creation and not changeable afterwards, so a wrong guess here means deleting and recreating.

## The three quiz answers

| Question | Answer |
|---|---|
| Each bucket has a default storage class, which you can specify when you create your bucket | **True** |
| An access control list (ACL) is a mechanism you can use to define who has access to your buckets and objects | **True** |
| You can stop publicly sharing an object by removing the permission entry that has | **allUsers** |

## Related files

- `gsp695-manage-storage-configuration-gsutil.md` for the same product at greater depth, including the transfer that has to be verified on both sides.
- `cloud-storage-three-bucket-lab.md` and `arc119-secure-data-lake-cloud-storage-challenge.md` for bucket work as a challenge.
- `pending-labs.md` for GSP421, the APIs Explorer version of these same operations.
