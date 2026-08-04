# GSP190, IAM Custom Roles

Nine tasks, six scored checkpoints, all gcloud in Cloud Shell. Fifteen minutes on paper, closer to eight in practice since every call returns instantly. Nothing is provisioned, so there are no waits at all.

Tasks 1 to 3 are read only exploration and unscored. The six checkpoints all sit in tasks 4 through 9.

Carries the "Do not deviate from instructions" warning. Everything below runs the lab's own commands, only replacing nano with a heredoc.

## No per run values except the region

The role ids (`editor`, `viewer`), titles, descriptions and permission lists are all fixed text from the lab. Only the region changes per run, and `$DEVSHELL_PROJECT_ID` covers the project. That makes this one of the few labs where the notes are the answer.

## Blocks matching the checkpoint boundaries

Region and tasks 1 to 3:

```
gcloud config set compute/region REGION

gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID | head -20
gcloud iam roles describe roles/viewer
gcloud iam list-grantable-roles //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID | head -20
```

Task 4, two checkpoints:

```
cat > role-definition.yaml <<'EOF'
title: "Role Editor"
description: "Edit access for App Versions"
stage: "ALPHA"
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
EOF

gcloud iam roles create editor --project $DEVSHELL_PROJECT_ID --file role-definition.yaml

gcloud iam roles create viewer --project $DEVSHELL_PROJECT_ID \
--title "Role Viewer" --description "Custom role description." \
--permissions compute.instances.get,compute.instances.list --stage ALPHA
```

Tasks 5 and 6, two checkpoints:

```
gcloud iam roles list --project $DEVSHELL_PROJECT_ID
gcloud iam roles list | head -20

gcloud iam roles describe editor --project $DEVSHELL_PROJECT_ID > new-role-definition.yaml
sed -i '/- appengine.versions.delete/a - storage.buckets.list' new-role-definition.yaml
sed -i '/- appengine.versions.delete/a - storage.buckets.get' new-role-definition.yaml
cat new-role-definition.yaml

gcloud iam roles update editor --project $DEVSHELL_PROJECT_ID \
--file new-role-definition.yaml

gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--add-permissions storage.buckets.get,storage.buckets.list
```

Task 7:

```
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID --stage DISABLED
```

Tasks 8 and 9:

```
gcloud iam roles delete viewer --project $DEVSHELL_PROJECT_ID --quiet
gcloud iam roles undelete viewer --project $DEVSHELL_PROJECT_ID
```

## Building the update yaml without nano

The lab has you run `describe`, copy the output out of the terminal by hand, open nano, paste it, and add two lines. Redirecting describe straight into the file does the same thing with no transcription:

```
gcloud iam roles describe editor --project $DEVSHELL_PROJECT_ID > new-role-definition.yaml
```

That output already contains the live `etag`, which is the point of the exercise. With the etag present the update applies silently. Without it gcloud asks whether you really want to overwrite, and a prompt inside a pasted block is how a whole block gets swallowed.

Two things about the sed:

- Insert **after** `- appengine.versions.delete`, not at the end of the file. Appending to the end puts the two lines after `title:` where they are no longer part of `includedPermissions`, and the yaml is then invalid.
- Two separate `sed -i` calls in reverse order rather than one with an embedded `\n`. Same result, nothing to get wrong about newline escaping. Listing `storage.buckets.list` first and `storage.buckets.get` second leaves them in the right order in the file.

`cat` the file before the update. Four permissions means the sed matched. Two means it did not, and the update will run and succeed while scoring nothing.

The describe output also carries a `name:` key, which gcloud accepts on update. The lab's own sample yaml includes it.

## Claim the disable checkpoint before deleting

Task 7 sets `viewer` to `stage: DISABLED`. Task 8 deletes it. Task 9 undeletes it, and the restored role comes back **still at `stage: DISABLED`**.

So after task 9 the role looks identical to how it looked after task 7. If the disable checkpoint has not been claimed by then there is no state left that distinguishes the two, and no way to redo task 7 without deviating. Get task 7 green, then run 8 and 9 together.

## Small things

`gcloud iam roles delete` prompts for confirmation. `--quiet` accepts it, which keeps the block paste safe.

`gcloud iam roles list` with no project flag dumps every predefined role in Google Cloud. Pipe it through `head` unless you want to scroll for a while.

Role ids are reusable only after 37 days once deleted, so a mistyped id cannot be corrected inside the lab. Both ids here are single words from the lab text, low risk, but worth knowing why it matters.
