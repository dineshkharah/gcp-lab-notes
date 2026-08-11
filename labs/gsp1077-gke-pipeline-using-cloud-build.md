# GSP1077, Google Kubernetes Engine Pipeline using Cloud Build

Nine tasks, **five** checkpoints, forty minutes stated and about thirty five in practice. Carries the "Do not deviate from instructions" warning.

Checkpoints: task 1 (services, Artifact Registry, cluster), task 3 (container image), task 4 (CI pipeline), task 5 (SSH key secret), task 6 (test environment and CD pipeline). **Tasks 7, 8 and 9 are unscored**, so 100 of 100 is reachable from Cloud Shell alone. The rollback in task 9 is one button in the console and earns nothing.

This run went clean start to finish following the lab text verbatim, which is rare enough to be worth saying. The notes below are the places it could have gone wrong and what to look at, not repairs.

Manual last updated and last tested July 2026, so the text matches the product.

## The shape

Two Git repositories on a self hosted **Gitea** server running on a VM called `git-server`, reachable on port 3000, credentials `giteaadmin` / `GiteaPassword123` passed inline in every remote url.

- `hello-cloudbuild-app` holds the Flask app, the Dockerfile and the CI config. One branch, `main`.
- `hello-cloudbuild-env` holds the Kubernetes manifest and the CD config. Two branches that matter: `candidate` is every deployment attempt, `production` is every successful one.

A push to `main` runs CI: test, build, push to Artifact Registry, then render `kubernetes.yaml.tpl` with the new image tag and commit it to `candidate`. A push to `candidate` runs CD: `kubectl apply` the manifest, and on success copy it to `production`.

In this version of the lab nothing is trigger driven. Every build is started by hand with `gcloud builds submit`, and the pushes exist only so the repositories hold the right history. That is why each task ends with both a `git push` and a `builds submit`.

## The git server ip is a variable that can come back empty

```
export GIT_SERVER_IP=$(gcloud compute instances describe git-server --zone=us-east4-c --format='get(networkInterfaces[0].accessConfigs[0].natIP)')
echo "[$GIT_SERVER_IP]"
```

The zone is hardcoded in the lab text. If the VM is ever placed elsewhere the describe fails, `GIT_SERVER_IP` is empty, and every remote silently becomes `http://giteaadmin:GiteaPassword123@:3000/...`. Brackets round the echo so an empty value is visible. The value is also printed on the lab panel as **Git Server IP**, so compare the two before pushing anything.

The same variable has to be re exported in any new tab. Nothing here needs a second tab, so the simplest answer is to stay in one.

## The two heredocs, and why the ip substitution is two steps

Tasks 6 writes both `cloudbuild.yaml` files with `cat <<'EOF'`, quoted, then substitutes afterwards:

```
sed -i "s/\${GIT_SERVER_IP}/$GIT_SERVER_IP/g" cloudbuild.yaml
```

The quoting is doing real work. Inside a quoted heredoc **nothing** expands, which is what keeps `$PROJECT_ID`, `$_SHORT_SHA` and `$(git log ...)` literal in the file so Cloud Build and the build step's own shell resolve them later. `${GIT_SERVER_IP}` is caught by the sed instead, because it is the one value that has to be baked in at write time.

Two things to verify after each heredoc, before pushing:

```
grep -n "3000" cloudbuild.yaml
tail -3 cloudbuild.yaml
```

The grep must show a real ip, not `${GIT_SERVER_IP}`. The tail must show the `options: logging: CLOUD_LOGGING_ONLY` block, which proves the paste reached `EOF`. A heredoc that gets truncated mid paste leaves the shell sitting at a `>` continuation prompt with a half written file, and the next command typed becomes part of the yaml.

`logging: CLOUD_LOGGING_ONLY` is not decoration. Without it a build that runs as a non default service account is rejected for having nowhere to write logs.

## Task 4's commit can have nothing to commit

```
git add .
git commit -m "Trigger CI pipeline"
```

Task 3 does not edit the working tree, so this only succeeds because `gcloud builds submit` drops a `.gcloudignore` in the directory on its first run. If that file already existed the commit exits non zero with `nothing to commit, working tree clean`. Harmless either way, the push and the build after it are what matter, but it looks like a failure in the middle of an otherwise clean block.

## The SSH key secret is vestigial and still scored

Task 5 generates a 4096 bit key and stores it in Secret Manager as `ssh_key_secret`. **Nothing in either pipeline reads it.** Both `cloudbuild.yaml` files authenticate to Gitea with the password in the url. The task is left over from the version of this lab that used SSH remotes, and it exists now purely because a checkpoint looks for the secret.

Do it anyway, and note the one prompt risk in the lab:

```
rm -f id_rsa id_rsa.pub
ssh-keygen -t rsa -b 4096 -N '' -f id_rsa -C "student@qwiklabs.net"
```

`-N ''` and `-f` between them suppress the passphrase and filename prompts, but **not** the overwrite prompt if the file already exists. Deleting first is cheaper than discovering that the way ARC114 did.

## Which service account actually runs the build

The lab grants `roles/container.developer` to `${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com`, the legacy Cloud Build agent. Newer projects run builds as the Compute Engine default service account instead, and where that is true the CD Deploy step fails on `kubectl apply` with a GKE permission error that reads like a cluster problem rather than an iam one.

If the Deploy step ever fails that way, grant the same role to the other account and resubmit:

```
gcloud projects add-iam-policy-binding $PROJECT_NUMBER \
  --member=serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com \
  --role=roles/container.developer
```

The Secret Manager grant in task 5 already targets the compute account, which is the hint that the two halves of the lab were written against different defaults.

## git pull inside a pasted block can stop for an editor

Task 6 and task 8 both run:

```
git pull http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-env.git candidate
```

Every time it is a fast forward, which needs no message and returns immediately. If it ever is not, git opens a merge message in the default editor and the rest of the pasted block queues up behind it. Two lines up front remove the possibility:

```
git config --global core.editor true
export GIT_MERGE_AUTOEDIT=no
```

## What rollback actually is, and which build to rebuild

Re executing an old CD build re runs it against **the source snapshot that build was submitted with**, which contains the older `kubernetes.yaml`, which names the older image tag. The cluster goes back to that image and the manifest gets copied to `production` again, so the production branch stays an honest history of what is deployed rather than of what was newest. That is the whole argument for the two repository split, and it is the one idea worth taking out of this lab.

The lab says to click the **second most recent** build. What you actually want is the **older of the two builds whose first step is `Deploy`**, that is, the earlier of the two builds submitted from `hello-cloudbuild-env`. Ordered newest first the history ends up:

1. CD, deployed Hello Cloud Build
2. CI, built Hello Cloud Build
3. CD, deployed Hello World, **this is the rollback target**
4. CI, built Hello World

Every build here came from `gcloud builds submit`, so the source column is an identical looking `gs://` tarball for all of them and the step list is the only way to tell CI from CD. CI builds have six steps starting with `Test`, CD builds have two starting with `Deploy`. Picking wrong is not destructive since task 9 is unscored, it just rebuilds an image and the page does not change back.

## Waiting for the endpoint

The service is a LoadBalancer, so task 8 needs a couple of minutes before the endpoint answers.

```
until [ -n "$(kubectl get svc hello-cloudbuild -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')" ]; do echo "waiting for external ip"; sleep 10; done; echo "IP READY"
curl -s http://$(kubectl get svc hello-cloudbuild -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

One physical line for the loop, for the reason in `gotchas.md`. `kubectl` needs credentials first if the shell was reconnected: `gcloud container clusters get-credentials hello-cloudbuild --region us-east4`.

The cluster is **regional** with `--num-nodes 1`, which means one node per zone and three nodes in us-east4, not one. `get-credentials` therefore takes `--region`, not `--zone`.

## Related files

- `gsp1079-continuous-delivery-cloud-deploy.md` and `gsp393-cicd-pipelines-challenge.md` solve the same problem with Cloud Deploy, where promotion between targets is a first class command instead of a git branch convention.
- `gsp1131-artifact-registry-qwik-start.md` for the `REGION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG` path anatomy used throughout.
- `gsp053-managing-deployments-kubernetes-engine.md` for what the deployment strategies behind this pipeline look like by hand.
