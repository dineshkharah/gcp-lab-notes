# GSP303, Configure Secure RDP using a Windows Bastion Host: Challenge Lab

Six checkpoints, thirty five minutes stated and **5 credits**. About twenty five minutes in practice, most of it Windows booting.

**The whole lab can be done from Cloud Shell, including the IIS task.** The lab presents task 3 as an RDP session plus GUI clicking in Server Manager, and it does not have to be. See the IIS section below, which is the only part of this lab worth reading twice.

Checkpoints: VPC network, VPC subnet, firewall rule, `vm-securehost`, `vm-bastionhost`, IIS.

Manual last updated February 2024, **lab last tested December 2023**.

## The task list at the top of the lab is from a different lab

It lists five "key tasks" including *"Create a virtual machine that points to the startup script"* and *"Configure a firewall rule to allow HTTP access to the virtual machine"*. Those are **GSP301's** tasks. The body of this lab has three tasks, mentions neither a startup script nor port 80, and has no checkpoints for either.

Ignore items 4 and 5. The concluding paragraph repeats the error, claiming you "configured a firewall rule to allow HTTP access", which you never do.

## Two NICs, and the syntax that forces

`--network-interface` **cannot be combined with `--network` or `--subnet`**, and each interface needs its own flag. Each NIC must also be in a **different** VPC, which the lab's design already satisfies.

```
gcloud compute instances create vm-securehost \
  --zone=$ZONE --machine-type=e2-medium \
  --image-family=windows-2016 --image-project=windows-cloud \
  --network-interface=subnet=securenetwork-subnet,no-address \
  --network-interface=subnet=default,no-address

gcloud compute instances create vm-bastionhost \
  --zone=$ZONE --machine-type=e2-medium \
  --image-family=windows-2016 --image-project=windows-cloud \
  --tags=allow-rdp-traffic \
  --network-interface=subnet=securenetwork-subnet \
  --network-interface=subnet=default,no-address
```

**Omitting `no-address` is what assigns the ephemeral external IP.** There is no positive flag for it.

**`--tags` goes on the bastion only.** The task says the rule "should be applied to the appropriate host using network tags", singular. The circulating script tags both hosts, which contradicts the entire scenario; it is harmless in effect since the secure host has no external address, but it is the thing that sentence is testing.

Verify before clicking, because a wrong interface is silent:

```
gcloud compute instances describe vm-securehost --zone=$ZONE --format="table(networkInterfaces[].subnetwork.basename(),networkInterfaces[].networkIP,networkInterfaces[].accessConfigs[0].natIP)"
```

`vm-securehost` must show `[None, None]` for NAT_IP.

## The network has to be custom mode

```
gcloud compute networks create securenetwork --subnet-mode=custom
gcloud compute networks subnets create securenetwork-subnet \
  --network=securenetwork --region=$REGION --range=192.168.16.0/20
gcloud compute firewall-rules create allow-rdp-ingress \
  --network=securenetwork --allow=tcp:3389 \
  --source-ranges=0.0.0.0/0 --target-tags=allow-rdp-traffic
```

Without `--subnet-mode=custom`, `networks create` auto-generates a subnet in every region and the subnet checkpoint has no distinct object to find. The firewall rule goes on **securenetwork**, not `default`.

## The image family is still there, and the -core variant is a trap

`windows-2016` was `READY` with a **July 2026** build, so the community script's pinned `windows-server-2016-dc-v20220513` is unnecessary and a hostage to that build being retired. Use the family.

Two families match: `windows-2016` is Datacenter, which **is** "Server with Desktop Experience", and `windows-2016-core` has no GUI. Picking core would leave the documented task 3 impossible, since there is no Server Manager.

A loop that falls forward if 2016 is ever retired, one physical line:

```
for f in windows-2016 windows-2019 windows-2022; do gcloud compute images describe-from-family $f --project=windows-cloud >/dev/null 2>&1 && export IMG_FAMILY=$f && break; done
```

## Password resets need the guest agent, so retry rather than sleep

`reset-windows-password` fails with `The instance may not be ready for use` until the guest agent is up, roughly ninety seconds to five minutes after create. It also **prompts for confirmation**, so `--quiet` is required inside a pasted block.

```
for i in $(seq 1 20); do gcloud compute reset-windows-password vm-bastionhost --user app_admin --zone=$ZONE --quiet && break; echo "guest agent not ready ($i), waiting 30s"; sleep 30; done
```

Three attempts in our run, about ninety seconds. The community script blocks for a flat five minutes instead, which is both slower and less reliable.

Note the lab's prose says the user is `app-admin` while its own command says `app_admin`. Use the underscore.

**Copy both passwords out of the terminal.** They are different per host and the command will not show them again.

## IIS without RDP, and the diagnostic that lied

This is the part worth keeping.

**What worked:** the startup script set **at instance creation**.

```
cat > iis.ps1 <<'EOF'
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
Write-Output "IIS_STATE=$((Get-WindowsFeature Web-Server).InstallState)"
Start-Service W3SVC
Write-Output "W3SVC=$((Get-Service W3SVC).Status)"
EOF

gcloud compute instances create vm-securehost \
  --zone=$ZONE --machine-type=e2-medium \
  --image-family=windows-2016 --image-project=windows-cloud \
  --network-interface=subnet=securenetwork-subnet,no-address \
  --network-interface=subnet=default,no-address \
  --metadata-from-file=windows-startup-script-ps1=iis.ps1
```

Wait about five and a half minutes, then click the checkpoint. It passed.

The IIS role installs from **local media**, not the internet, which is why this works on a machine with no external connectivity at all.

**What did not work:** `add-metadata` with `windows-startup-script-ps1` followed by `instances reset`. Tried twice, both times the checkpoint stayed at 0 of 15. Set the metadata at **create** time.

Since checkpoint 4 latches once earned, **deleting and recreating `vm-securehost` costs nothing**, which is what makes the switch to a creation-time script free.

**The serial console was useless here, and worse than useless.** All three attempts, including the successful one, produced **no matching output at all**:

```
gcloud compute instances get-serial-port-output vm-securehost --zone=$ZONE | grep -i -E "IIS_STATE|W3SVC|startup-script"
```

Empty every time. I concluded from that silence that the script was not running, twice, and was wrong. `Write-Host` versus `Write-Output` was a red herring; neither appeared. On these Windows images the startup script runner's output does not reach the serial buffer that `get-serial-port-output` returns.

So on a Windows instance with no external IP, **the cheapest test of whether the script worked is clicking the checkpoint**, not reading logs.

## Why the second NIC on default exists

The scenario calls it a monitoring requirement and it reads like flavour text. It is not.

`lab-monitor`, the pre-created instance you are told not to modify, sits on the **default** network at `10.138.0.2`. Your secure host's second NIC is on default too. And default carries:

```
default-allow-internal   tcp:0-65535,udp:0-65535,icmp   10.128.0.0/9
```

So `lab-monitor` can reach the secure host on port 80 over the default network. Since the secure host has no external address, that is almost certainly how the IIS checkpoint sees it. Inference rather than proof, but it explains why the second NIC is mandatory and why `lab-monitor` must be left alone.

## The RDP path, if it is ever needed

Kept because the checkpoint was written around it, and because the startup script route could stop working.

1. `mstsc` to the bastion's external IP. Username **`.\app_admin`** — the `.\` forces a local account, without it Windows 11 may offer your own account and report "the credentials did not work" with a correct password. Accept the certificate warning.
2. Inside the bastion, `mstsc` to the secure host's **`securenetwork-subnet`** address, with that host's own password.
3. Server Manager → **Manage** → Add Roles and Features → Role-based or feature-based → your server → tick **Web Server (IIS)** → **Add Features** on the prompt → Next through → **Install**.
4. `http://localhost` on the secure host to confirm.

The risk the lab admits to: **many networks block outbound 3389**, and its only advice is to use a different network. A **timeout** to the bastion is your own network; **credentials did not work** is the missing `.\`.

## Related files

- `gsp301-remote-startup-script-challenge.md` for startup scripts on Linux, and the source of the stale task list items pasted into this lab.
- `gsp322-build-a-secure-network-challenge.md` for the same bastion pattern on Linux, with IAP instead of a public RDP port.
