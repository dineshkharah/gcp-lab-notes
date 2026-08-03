# GSP1048, Cloud Spanner Database Fundamentals

Seven tasks: create an instance, a database, a table, insert and query data, do the same again from the cli, do it a third time with Terraform, then delete one instance. Billed as ten minutes, closer to twenty because three Spanner instances get created and Terraform has to be installed.

Three scored checkpoints: instance and database, the schema, and the instance and database made with the cli. Nothing is scored after that.

## Tasks 1 to 4, written as console clicking but all gcloud

```
export REGION=YOUR_REGION
export PROJECT_ID=$DEVSHELL_PROJECT_ID

gcloud spanner instances create banking-instance \
  --config=regional-$REGION \
  --description="banking-instance" \
  --nodes=1 \
  --edition=ENTERPRISE

gcloud spanner databases create banking-db --instance=banking-instance

gcloud spanner databases ddl update banking-db --instance=banking-instance \
  --ddl='CREATE TABLE Customer (
    CustomerId STRING(36) NOT NULL,
    Name STRING(MAX) NOT NULL,
    Location STRING(MAX) NOT NULL
  ) PRIMARY KEY (CustomerId)'

gcloud spanner databases execute-sql banking-db --instance=banking-instance \
  --sql="INSERT INTO Customer (CustomerId, Name, Location) VALUES
    ('bdaaaa97-1b4b-4e58-b4ad-84030de92235', 'Richard Nelson', 'Ada Ohio'),
    ('b2b4002d-7813-4551-b83b-366ef95f9273', 'Shana Underwood', 'Ely Iowa')"

gcloud spanner databases execute-sql banking-db --instance=banking-instance \
  --sql='SELECT * FROM Customer'
```

The lab asks for the Enterprise edition in the console form. `--edition=ENTERPRISE` covers it on recent gcloud versions, and Enterprise is the default anyway, so drop the flag if your version rejects it. The checkpoint only looks for the instance and database.

The lab has you insert the two rows as separate statements. One statement with two value tuples is the same result.

## Task 5, the same thing from the cli

```
gcloud spanner instances create banking-instance-2 \
  --config=regional-$REGION \
  --description="Banking Instance 2" \
  --nodes=2

gcloud spanner databases create banking-db-2 --instance=banking-instance-2

gcloud spanner instances list

gcloud spanner instances update banking-instance-2 --nodes=1
gcloud spanner instances list
```

## Task 6, Terraform

Installing it is the actual lesson here, so keep this part as the lab writes it. The `.customize_environment` file is what makes the install survive a Cloud Shell restart.

```
cat <<'EOF' > ~/.customize_environment
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
EOF
bash ~/.customize_environment

terraform --version
```

Then the config. The lab's snippet has no `provider` block, which leaves the project to be picked up from the environment. Adding it makes the project explicit.

```
cat > spanner.tf <<EOF
provider "google" {
  project = "$PROJECT_ID"
}

resource "google_spanner_instance" "banking-instance-3" {
  name         = "banking-instance-3"
  config       = "regional-$REGION"
  display_name = "Banking Instance 3"
  num_nodes    = 2
  labels = {
  }
}
EOF

terraform init
terraform plan
terraform apply -auto-approve

gcloud spanner instances list
```

`-auto-approve` skips the interactive yes. Drop it if you want to see the plan pause as the lab shows.

## Task 7, delete

Run this only after the task 5 checkpoint has gone green, because it removes the instance that checkpoint looks at.

```
gcloud spanner instances delete banking-instance-2 --quiet
gcloud spanner instances list
```

Finishes with `banking-instance` and `banking-instance-3` left in place. The lab does not ask for any more cleanup.

## Name collision worth watching

This lab uses `banking-instance` and `banking-db`. GSP1050 and the GSP381 challenge lab use `banking-ops-instance` and `banking-ops-db`. Easy to mix up when running them back to back.
