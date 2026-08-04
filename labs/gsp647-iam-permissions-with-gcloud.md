# GSP647, Configuring IAM Permissions with gcloud

Seven tasks, **twelve** scored checkpoints, the most of any lab in the Privileged Access with IAM badge. Twenty five minutes on paper, more like thirty five to forty in practice.

Two user accounts and two projects. User 1 owns both projects, user 2 starts as viewer on project 1 only. The lab is about `gcloud config configurations`, so the spine of it is switching identity back and forth.

Carries the "Do not deviate from instructions" warning.

## It does not run in Cloud Shell

Everything happens inside a browser SSH session on a prebuilt vm called `centos-clean`, reached from Compute Engine, VM instances, SSH. The vm is pretending to be a machine without gcloud installed. Cloud Shell is never opened.

Consequences worth knowing before starting:

- No `$DEVSHELL_PROJECT_ID`. Both project ids have to be handled explicitly.
- The `SA` variable in task 5 is a plain shell variable, not in `.bashrc`, so tasks 5 and 6 must run in one unbroken session.
- Restarting the SSH window loses `SA` but keeps `PROJECTID2` and `USERID2`, since those go into `.bashrc`.

## Two zones in two different regions

Read both zones off the lab page and do not assume they share a region. In our run zone 1 was `us-east4-b` and zone 2 was **`us-east1-c`**. Only `lab-1` goes in zone 1; `lab-2`, `lab-3` and `lab-4` all go in zone 2.

## Values needed up front

Username 1, username 2, project id 1, project id 2, region, zone 1, zone 2. Both usernames share one password.

## The two interactive auth round-trips

Neither can be scripted. Together they are five to ten minutes of the runtime.

**`gcloud auth login`** for user 1: ENTER at the continue prompt, open the link, pick user 1, Allow, paste the verification code back.

**`gcloud init --no-launch-browser`** for user 2:

| Prompt | Answer |
| --- | --- |
| Pick configuration to use | 2, Create a new configuration |
| Enter configuration name | `user2` |
| Select an account | the entry reading **Sign in with a new Google Account** |
| Do you want to continue (Y/n) | ENTER |
| Pick cloud project to use | the number beside **project id 1** |

Pick the account by its label, not by number. The lab text says option 3, and it was 3 in our run, but only because the list happened to be the compute service account, user 1, sign in with a new account, skip this step. A different list order silently authenticates the wrong identity.

In the browser, click **Use another account** before entering user 2's address. Landing on user 1's consent screen and clicking through it creates the `user2` configuration on the wrong account.

Project id 2 is **not** in the project list at this point, which is the premise of the lab rather than a problem.

Init then sets user 2's region and zone from project metadata, so it does not prompt for them.

## The gotcha that cost a recheck: the scorer reads the active configuration

The "Update the default zone" checkpoint went 0/10 with only `Please change the zone as instructed in the lab manual` to go on, despite `config_default` correctly holding `zone = us-east4-a`.

Cause: `gcloud init` leaves the **newly created** configuration active. The zone change had been made on `default`, but by the time the checkpoint was clicked, `user2` was active with `us-east4-b`.

```
gcloud config configurations activate default
gcloud config configurations list
```

`configurations list` is the useful command here. It prints every configuration with its account, project and zone in one table, so which one is active and what each holds is visible at a glance. Then the recheck passed with nothing else changed.

General form of this: **before clicking any checkpoint in this lab, confirm which configuration is active.** Half the failures in a two-identity lab are the right command run as the wrong principal.

## Default has no project, and lab-1 still works

`gcloud auth login` does not set a project, so `default` shows an empty project column all lab. `gcloud compute instances create lab-1` succeeds anyway because gcloud running on a Compute Engine vm falls back to the metadata server's project.

Harmless, but do not read the empty column as something to fix. Every scored command afterwards either passes `--project` or sets the project first.

## Three deliberate permission failures

The lab has you attempt `gcloud compute instances create lab-2` three times, and the first two are supposed to fail:

1. as user 2 in project 1, before any grant
2. as user 2 in project 2, after the viewer grant but before the devops role
3. as user 2 in project 2, after the devops role, which is the scored one

Each failure takes 30 to 60 seconds to come back, and the lab itself notes "it takes a little time to fail". The error names the zone and project, which is the quickest way to confirm which attempt you are looking at.

If the third one fails too, wait sixty seconds and run it again. Custom role bindings take a moment to propagate, and by that point three identical looking errors have gone by and it is easy to assume something is actually wrong.

## Echo every variable before using it

`PROJECTID2` and `USERID2` are appended to `.bashrc` with `>>`, and `SA` is captured from a filtered list. All three feed straight into `add-iam-policy-binding`, where an empty value produces an `INVALID_ARGUMENT` rather than anything readable.

```
echo $PROJECTID2
echo $USERID2
echo $SA
```

`echo $SA` should print `devops@PROJECT_ID_2.iam.gserviceaccount.com`.

The `>>` also means running the append block twice leaves duplicate export lines. Same value twice is harmless, but a mistyped value stays in the file even after a correction.

## Answer N once, on purpose

Task 3 has user 2 try to set the project to project 2 before the grant exists:

```
gcloud config set project $PROJECTID2
```

It warns about missing access and asks `Do you want to continue (Y/n)?`. The lab wants **N**. Answering Y sets the property anyway and puts the configuration into a state the rest of the lab does not expect.

That prompt is also a reason to run this command on its own rather than inside a pasted block.

## The scored sequence, condensed

| # | Checkpoint | Run as |
| --- | --- | --- |
| 1 | lab-1 in project 1 | default |
| 2 | default zone updated | default, and default must be active when checked |
| 3 | user2 configuration created | either |
| 4 | user2 restricted to viewer on project 2 | default |
| 5 | devops custom role created | default |
| 6 | user2 bound to iam.serviceAccountUser | default |
| 7 | user2 bound to devops role | default |
| 8 | lab-2 in project 2 | user2 |
| 9 | devops service account created | default, project set to project 2 |
| 10 | service account bound to iam.serviceAccountUser | default |
| 11 | service account bound to compute.instanceAdmin | default |
| 12 | lab-3 has the service account attached | default |

Only checkpoint 8 is claimed as user 2. Everything else is default.

## Task 7 is unscored

`gcloud compute ssh lab-3 --zone ZONE2`, ENTER at the continue prompt, ENTER twice for an empty passphrase. Inside, `gcloud config list` shows the account as the devops service account rather than a student address, and `gcloud compute instances create lab-4` then works with no human authentication anywhere. That is the point of the task, but the score is already complete before it.

## Quiz answers

- Override the configured zone once: `--zone DIFFERENT_ZONE`
- Two of three items needed to bind a role to a project: **project id** and **account**
- Not true about service accounts: *service accounts always provide full admin rights to the project*
