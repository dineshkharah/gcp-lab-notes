# GSP1131, Artifact Registry: Qwik Start

Five tasks, two checkpoints, ten minutes. All Cloud Shell, no problems in our run. Docker is already installed and authenticated in Cloud Shell, so there is nothing to set up.

Checkpoints: **Create a Docker repository** after task 1, and **Add the image to the repository** after task 4.

## One value to substitute

Only the region changes per run. It appears in four separate commands, and every one of them has to match the region the repository was created in, or the push goes to a hostname that does not host your repository.

Setting it once removes the risk:

```
export PROJECT_ID=$(gcloud config get-value project)
export REGION=YOUR_REGION
export REPO=example-docker-repo
export IMAGE=$REGION-docker.pkg.dev/$PROJECT_ID/$REPO/sample-image:tag1
echo $IMAGE
```

`echo $IMAGE` before pushing. It should read like `us-central1-docker.pkg.dev/qwiklabs-gcp-nn-xxxx/example-docker-repo/sample-image:tag1`.

## Task 1, create the repository

```
gcloud artifacts repositories create $REPO --repository-format=docker \
    --location=$REGION --description="Docker repository" \
    --project=$PROJECT_ID

gcloud artifacts repositories list --project=$PROJECT_ID
```

The name `example-docker-repo` and the description `Docker repository` are both dictated by the lab. Claim the first checkpoint here.

## Task 2, docker auth

```
gcloud auth configure-docker $REGION-docker.pkg.dev
```

**This prompts.** It asks whether to add the registry to the Docker credential helper config and waits for `Y`. Keep it out of a longer paste, or the next line of the paste answers it. Same failure mode as the `read` trap in `arc114-speech-and-language-challenge.md`.

This writes the credential helper into `~/.docker/config.json`; it authenticates the region host, not a single repository.

## Tasks 3 to 5, pull, tag, push, pull

```
docker pull us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0

docker tag us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0 $IMAGE

docker push $IMAGE

docker pull $IMAGE
```

The source image is public and lives at `us-docker.pkg.dev`, which is **not** your region's host. Only the tag target uses `$REGION`. Mixing those up is the one easy mistake here.

Claim the second checkpoint after the push. The final `docker pull` is the lab's demonstration and reports `Image is up to date`, since the layers are already local from the tag.

## Reading an Artifact Registry path

Worth internalising, since every later Docker lab uses this shape:

```
LOCATION-docker.pkg.dev / PROJECT_ID / REPO_ID / IMAGE_PATH : TAG
```

For the sample image, `us-docker.pkg.dev` is the host, `google-samples` the project, `containers` the repository, `/gke/hello-app` the path within it, `1.0` the tag. Omitting the tag makes Docker default to `latest`, which is not what the checkpoint wants here.
