# GSP191, Modular Load Balancing with Terraform, Regional Load Balancer

Two tasks, **one** checkpoint, ten minutes stated and **5 credits**. **Entirely Cloud Shell.** The Terraform work is about five minutes; the whole attempt took longer because of one environment trap described below.

**Manual and last tested both July 19 2024**, the stalest dates of any lab in these notes.

Smaller sibling of `gsp206-https-content-based-load-balancer-terraform.md`: same module family, one regional TCP load balancer instead of a global HTTPS one, **10 resources rather than 43**.

## The trap: an installed Terraform that still is not the one you run

**This lab has no install step**, because in July 2024 Cloud Shell shipped Terraform. It no longer does, and what it ships instead is a **stub named `terraform` that prints installation instructions**:

```
  Follow the instructions at https://developer.hashicorp.com/terraform/install to install terraform, or run the commands below:
  ...
  To persist the installation between sessions, either add the above commands to $HOME/.customize_environment
```

Two ways that bites, and this run hit both.

**`command -v terraform` is a false positive.** The guard `command -v terraform >/dev/null || bash ~/.customize_environment` found the stub, concluded Terraform was present, and skipped the install entirely. Every subsequent `init`, `plan`, `apply` and `output` printed the stub's help text and returned success.

**And installing it does not fix the current shell.** Running the install unconditionally afterwards worked, `Setting up terraform (1.15.9-1)`, and `terraform --version` **still printed the stub**. Either the stub sits earlier in `PATH` than `/usr/bin`, or bash had cached the stub's location from the earlier lookup, which it does for every command it resolves. Both are plausible and the remedies differ:

```
type -a terraform          # shows every terraform on PATH, in order
hash -r                    # clears bash's cached command locations
/usr/bin/terraform --version
```

What actually worked was **bypassing the name entirely**:

```
export TF=/usr/bin/terraform
$TF --version
$TF init -input=false
$TF plan -input=false -out=tfplan && $TF apply tfplan
```

**Verify by content, not by presence.** The check that would have caught this in the first minute:

```
terraform --version | grep -q '^Terraform v' && echo "TERRAFORM OK" || echo "TERRAFORM STILL NOT INSTALLED"
```

That line printed `TERRAFORM STILL NOT INSTALLED` immediately after a successful apt install, which is exactly the information needed.

**The failure cascades and hides itself.** Because the stub exits 0 and prints to stdout, `terraform output -json` returned its help text, the `python3 json.load` parsing it raised a `JSONDecodeError`, `EXTERNAL_IP` came out empty, and `curl` then complained `option -: is unknown` because the words `wget -O -` from the stub message had landed on its command line. Four unrelated looking errors, one cause, and none of them mentioning Terraform not being installed.

## The variable is project_id, not project

Every snippet in the lab's own overview text uses `"${var.project}"`, but the example's `variables.tf` declares:

```
17:variable "region" {
21:variable "project_id" {
25:variable "image_family" {
30:variable "image_project" {
```

So **`TF_VAR_project_id`** is the one that answers the prompt. Read the file rather than the prose:

```
grep -n '^variable' variables.tf
```

Setting both `TF_VAR_project_id` and `TF_VAR_project` is harmless, since an undeclared `TF_VAR_` is ignored, and it means the prompt is answered whichever name the example uses.

## The working commands

```
bash ~/.customize_environment
export TF=/usr/bin/terraform
$TF --version
export REGION=us-east4
export GOOGLE_PROJECT=$(gcloud config get-value project)
export TF_VAR_project_id=$GOOGLE_PROJECT
export TF_VAR_project=$GOOGLE_PROJECT
export TF_VAR_region=$REGION
cd ~ && rm -rf terraform-google-lb && git clone -q https://github.com/GoogleCloudPlatform/terraform-google-lb
cd ~/terraform-google-lb/examples/basic
grep -n '^variable' variables.tf
sed -i 's/us-central1/'"$REGION"'/g' variables.tf
grep -n "$REGION" variables.tf
$TF init -input=false
$TF plan -input=false -out=tfplan && $TF apply tfplan
$TF output
export EXTERNAL_IP=$($TF output -json | python3 -c "import sys,json; d=json.load(sys.stdin); print(next((str(v['value']) for k,v in d.items() if 'ip' in k.lower()), ''))")
echo "EXTERNAL_IP=[$EXTERNAL_IP]"
```

`Apply complete! Resources: 10 added` is what the single checkpoint reads.

The `sed` is the lab's own and is safe here because `us-central1` really is the default in `variables.tf`. Confirm with the `grep` afterwards rather than assuming.

**`plan -out=tfplan && apply tfplan`** rather than `apply -auto-approve`. Applying a saved plan prompts for nothing, so there is no prompt for a stray input to land in, and the `&&` stops a failed plan from being followed by an apply.

Then the verification, which is the lab's *"refresh a few times"* step done from the shell:

```
for i in $(seq 1 20); do C=$(curl -s -o /dev/null -w '%{http_code}' -m 10 "http://$EXTERNAL_IP/"); echo "lb HTTP $C"; [ "$C" = "200" ] && break; sleep 15; done
for i in $(seq 1 12); do curl -s -m 10 "http://$EXTERNAL_IP/" | grep -oiE '(group1|vm|instance)[-a-z0-9]*' | head -1; sleep 2; done | sort | uniq -c
```

Two distinct instance names in that count is the target pool balancing across the managed instance group's two instances.

## Two years stale, and it still works

Worth recording because the prediction was wrong. The example is Terraform 0.11 era, full of `"${var.region}"` interpolation and `backends = { "0" = [...] }` map syntax, run against **Terraform 1.15.9** and modules cloned from upstream HEAD. It planned and applied without complaint.

So a stale manual date is a reason to expect trouble, not evidence of it. The thing that actually broke was the environment the lab assumes, not the code it ships.

## The community helper

Its one genuinely correct detail is `TF_VAR_project_id`, which matches the file. Four problems, one of them serious:

- **`yes | terraform apply --auto-approve`** is self defeating. `--auto-approve` removes the approval prompt, so the only prompt left is an unanswered variable, and `yes` fills it with the string `y`. If its `TF_VAR_` had not matched, Terraform would have gone looking for a project literally named `y`. Its `terraform plan` at the previous step has nothing piped in and would simply hang.
- **The region comes from `gcloud config get-value compute/region`**, not the lab page. Fifth helper in these notes to take an assigned value out of the environment; if that config is unset it drops into an interactive region picker mid script.
- **No Terraform install**, which is the one thing this lab genuinely needs adding.
- **No verification at all.** It never captures the load balancer IP, so the lab's own closing step cannot be done from where it leaves you.

## Related files

- `gsp206-https-content-based-load-balancer-terraform.md`, the global HTTPS version, with the `.customize_environment` install and the line number trap.
- `gsp528-ncc-challenge.md` and `gsp399-design-implement-network-security-challenge.md` for Google Cloud load balancing and networking without Terraform.
- `gotchas.md`, the section on a command existing not meaning the tool is installed, which came from this lab.
