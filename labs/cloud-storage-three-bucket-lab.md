# Cloud Storage challenge lab, three buckets

A short form based challenge lab. Three tasks: create one bucket, make an object in a second bucket public, add labels to a third. Five minutes.

The three bucket names are handed to you and two of the buckets already exist.

## Get the real bucket names from the project, not the panel

The bucket name fields in the credentials panel are clipped. In this run the naming looked inconsistent, with two buckets using a hyphen before a suffix and the first apparently not. Listing the pre created ones showed the pattern for two of them:

```
gcloud storage ls
```

The first bucket did not exist yet so it could not be listed, and the pattern guess was wrong. The scorer message gave the real name: `Please create a bucket 'NAME' as specified in lab instructions`. Trust the scorer message over the pattern.

## Task 1

```
gcloud storage buckets create gs://BUCKET1 --location=YOUR_REGION
```

## Task 2, public object

The task is explicit that the grant goes on the **object**, not the bucket. A bucket level iam grant does not pass.

The pre created bucket has uniform bucket level access turned on, which forbids per object acls entirely, so it has to be turned off first:

```
gcloud storage buckets update gs://BUCKET2 --no-uniform-bucket-level-access

gcloud storage objects update gs://BUCKET2/sample.txt \
  --add-acl-grant=entity=AllUsers,role=READER
```

Running both in one paste failed, because the object update went out before the bucket setting had taken effect. Run the update, confirm `uniform_bucket_level_access` reads False, then do the object grant.

## Task 3, labels

No specific keys are required, any valid pair works.

```
gcloud storage buckets update gs://BUCKET3 --update-labels=env=dev,team=operations
gcloud storage buckets describe gs://BUCKET3 --format="value(labels)"
```
