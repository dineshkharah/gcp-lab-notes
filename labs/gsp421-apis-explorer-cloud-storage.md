# GSP421, APIs Explorer: Cloud Storage

Six tasks plus a quiz, **five** checkpoints, fifteen minutes stated and **no cost**. About **ten minutes**. **Browser lab, zero Cloud Shell**: the APIs Explorer web form plus one console upload.

Manual and last tested both April 2026.

The same operations as `gsp074-cloud-storage-qwik-start-cli-sdk.md`, done through the JSON API's form instead of `gcloud storage`. Useful read against each other, since the form makes visible what the CLI hides: `buckets.insert` takes `project` as a query parameter and `name` in the request body, and `objects.copy` names four separate fields where `gcloud storage cp` takes two paths.

## The files

`labs/assets/GSP421-demo-image1.png` and `labs/assets/GSP421-demo-image2.png`, the dog and the Ada Lovelace portrait, saved so the download step off the lab page can be skipped next time.

`GSP421-demo-image1.png` is **401,951 bytes**, which is exactly the `"size": "401951"` in the lab's own sample `objects.copy` response. Handy confirmation that the image the lab links today is the one its expected output was written against.

## The ordering trap, three checkpoints deep

**Tasks 5 and 6 destroy what tasks 3 and 4 score.** Task 5 deletes both images out of bucket 1, and task 6 deletes bucket 1 entirely.

So the claim order is not optional:

| Do this | Then this |
|---|---|
| task 3, both images uploaded | **click checkpoint 3** |
| task 4, the copy in bucket 2 | **click checkpoint 4** |
| task 5, both deletes | click checkpoint 5 |
| task 6, delete bucket 1 | |

Checkpoint 4 survives task 6 on its own, since the copy lives in bucket 2 and only bucket 1 is deleted. Checkpoint 3 does not survive task 5 at all.

## How task 4 was actually done

The lab routes task 4 through `objects.copy` with four fields, `sourceBucket`, `sourceObject`, `destinationBucket`, `destinationObject=demo-image1-copy.png`.

This run instead **uploaded `demo-image1.png` into bucket 2 through the console and renamed it** to `demo-image1-copy.png` with the object's three dot menu. Checkpoint 4 passed.

Worth recording because it tells you which kind of scorer this is. The checkpoint reads for an object named `demo-image1-copy.png` in the second bucket and does not care how it got there, which puts it in the read state camp rather than the watch for an operation camp that GSP510 turned out to be. See the scorer wants the action section in `gotchas.md` for the distinction, and note that there is no way to know which you have in advance.

## The form level traps

Four things, all of which the lab repeats on every task, which is itself the signal that they cost people time:

- **Both Credentials checkboxes**, Google OAuth 2.0 **and** API key, have to be ticked. This is the usual cause of a 401 that reads as a permissions problem rather than a missing box.
- **No trailing space in the `project` field.** A pasted project id frequently carries one and the error does not say so.
- **`buckets.delete` requires an empty bucket.** Bucket 1 only becomes empty after task 5, so task 6 run early returns a 409.
- **Bucket names are globally unique.** Prefix with the project id and the collision cannot happen.

## Two responses worth reading rather than skimming

`buckets.insert` with only `name` in the body returns `"location": "US"` and `"locationType": "multi-region"`. Nothing in the request asked for that; it is the default, and it is what the lab expects. Do not add a location trying to match a region.

The delete methods return **`204`** and no body. A green 204 ribbon is success, not an empty failure, which is worth knowing before staring at a blank response pane.

## Task 7, the quiz

| Question | Answer |
|---|---|
| Each bucket has a default storage class, which you can specify when you create your bucket | **True** |
| Every bucket must have a unique name across the entire Cloud Storage namespace | **True** |
| Cloud Storage offers four storage classes | **Coldline Storage, Multi-Regional Storage, Regional Storage, Nearline Storage** |

`Local storage` and `Cross region storage` are not storage classes.

## Related files

- `gsp074-cloud-storage-qwik-start-cli-sdk.md`, the CLI version of the same operations, including the same shape of ordering trap.
- `gsp695-manage-storage-configuration-gsutil.md` for the same product at depth.
- `arc109-api-gateway-challenge.md` and `gsp872-api-gateway-qwik-start.md` for Google APIs from the other direction, publishing rather than calling.
