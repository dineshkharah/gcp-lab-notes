# ARC111, Implement Cloud Storage and Data Protection Solutions: Challenge Lab

Three tasks, **three** checkpoints, fifteen minutes stated and **no cost**. About **five minutes**. Doable entirely in Cloud Shell or entirely in the console.

Manual last updated April 2026 but **lab last tested June 2024**, the oldest test date in these notes. Nothing broke because of it, but treat odd error messages as expected rather than as your mistake.

This file was originally written as `cloud-storage-three-bucket-lab.md` from a run that never recorded the lab id. Renamed once ARC111 turned up again.

## The task set is randomised

The lab says so: *"You've been assigned random tasks from the set of tasks"*, with a **Form ID** to quote when reporting issues, `form-2` on this run. So another run can hand you different tasks under the same lab id. The three below are what form-2 asks for, and **the tasks are explicitly independent**, so they can be done in any order.

| Task | What it wants |
|---|---|
| 1 | create a bucket named exactly Bucket1 |
| 2 | grant `allUsers` READER on the **object** inside Bucket2 |
| 3 | add labels to Bucket3 |

## Read the bucket names verbatim, do not infer the pattern

The three names come off the lab panel and **they are not consistently formed**. From this run:

```
Bucket1  qwiklabs-gcp-03-1c060b285e24e2xz-bucket
Bucket2  qwiklabs-gcp-03-1c060b285e24-e2xz-gcs-bucket
Bucket3  qwiklabs-gcp-03-1c060b285e24-e2xz-bucket-ops
```

Bucket1 has **no hyphen** before the random suffix while the other two do. An earlier run of this lab assumed that was the panel clipping a character, guessed the consistent form, and got it wrong. It is not clipping, the name really is irregular, and only Bucket1 is affected because it is the one you have to type rather than the one you can list.

Two consequences. **Copy Bucket1 out of the panel character for character**, and if the checkpoint still fails, the scorer message hands you the answer outright: `Please create a bucket 'NAME' as specified in lab instructions`. Trust that string over any pattern.

## The commands, in two blocks

```
gcloud config set compute/region us-west1
export B1=qwiklabs-gcp-03-1c060b285e24e2xz-bucket
export B2=qwiklabs-gcp-03-1c060b285e24-e2xz-gcs-bucket
export B3=qwiklabs-gcp-03-1c060b285e24-e2xz-bucket-ops
gcloud storage buckets create gs://$B1 --location=us-west1
gcloud storage buckets update gs://$B3 --update-labels=env=dev,team=operations
gcloud storage buckets describe gs://$B3 --format="value(labels)"
gcloud storage buckets update gs://$B2 --no-uniform-bucket-level-access
gcloud storage buckets describe gs://$B2 --format="value(uniform_bucket_level_access)"
gcloud storage ls gs://$B2
```
Claims checkpoints 1 and 3. Two lines to read before continuing: `uniform_bucket_level_access` must print **False**, and the `ls` gives the object name for the next block, `sample.txt` on the earlier run.

```
gcloud storage objects update gs://$B2/sample.txt --add-acl-grant=entity=AllUsers,role=READER
curl -s -o /dev/null -w "public HTTP %{http_code}\n" https://storage.googleapis.com/$B2/sample.txt
```
Claims checkpoint 2. `public HTTP 200` is the proof.

Task 3 accepts **any valid label pair**; no keys are specified anywhere in the task.

## Why task 2 has to be two blocks

**The pre created Bucket2 has uniform bucket level access enabled, and that forbids per object ACLs outright.** So the ACL grant cannot be the first thing you do.

Running the bucket update and the object grant in a single paste **failed on the earlier run**, because the object update went out before the bucket setting had taken effect. Splitting them is the entire fix. Confirm `False`, then grant.

This is the shape of the propagation problem recorded elsewhere in `gotchas.md` under a binding that exists is not yet a binding that works, arriving from the other direction: here the setting has to *stop* applying before the next call is legal.

## The grant is on the object, not the bucket

The task says it twice, in the body and again in the note: *"Instead of a bucket, grant all users read permissions to the object."*

A bucket level IAM binding for `allUsers` with `roles/storage.objectViewer` makes the object publicly readable and **does not pass this checkpoint**. The scorer reads the object's ACL. This is the one place in the lab where a technically correct solution scores zero, which is why the task repeats itself.

## The console route

Done in the console on this run, and it hits the same constraint in a different costume. Bucket2's **Permissions** tab shows Access control as **Uniform**, and the per object controls are simply not offered until it is switched to **Fine-grained**. That switch is the console's equivalent of `--no-uniform-bucket-level-access`, and the object's own access editing only appears afterwards.

Task 3's labels live under the bucket's **Configuration** tab rather than anywhere in Permissions, which is worth knowing since labels read like an access concept and are not.

## Related files

- `gsp074-cloud-storage-qwik-start-cli-sdk.md` for the same public object ACL on a bucket without uniform access enabled, where no preparation step is needed.
- `gsp421-apis-explorer-cloud-storage.md` for the same operations through the JSON API form.
- `arc119-secure-data-lake-cloud-storage-challenge.md` and `gsp695-manage-storage-configuration-gsutil.md` for Cloud Storage at more depth.
