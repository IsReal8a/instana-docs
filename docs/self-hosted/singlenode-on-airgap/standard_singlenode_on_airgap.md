---
title: Standard Single-node on airgap
parent: Self-hosted
nav_order: 3
---

# Standard Single-node on airgap
{: .no_toc }

Technical guide on how to install the Instana backend single-node on an "airgapped" environment.

Contribution made by: [Sergey Omelaenko, Solution Consultant, Turbonomic ARM at IBM](https://www.linkedin.com/in/sergeyomelaenko).

Edited by: Israel Ochoa.

Based on Instana build **294**.

Updated: 23 April 2025
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Purpose of this guide and pre-requisites

{: .important }
> The purpose is to teach you step by step how to install Instana Standard edition single-node on an "airgapped" environment, please consider this as a cheatsheet and use it as an example, you need to adjust this guide to your environment(s) and needs.

{: .warning }
> Always read the official documentation before proceeding:

[Single Node Documentation](https://www.ibm.com/docs/en/instana-observability/1.0.297?topic=edition-single-node-cluster){: .btn .btn-blue}

## System requirements

```shell
Red Hat® Enterprise Linux® (RHEL) 9 and 8 x86_64
CentOS Stream 9 x86_64

24 CPU cores
96GB MEM
200GB for boot disk
6TB for data, metrics, analytics, etc. 
```

## How to read this guide?

Since we're going to work on a host without Internet connection aka **Instana Backend** and a Bastion host with Internet connection (to artifact-public.instana.io) aka **Bastion**, the guide indicates where to work for each step.

## Instana Backend

On Instana backend host.
Check `dev` name for the 6TB volume:

```shell
lsblk
```

e.g `/dev/nvme0n2`

Make dir and mount the volume for Instana data:

```shell
mkdir -p /mnt/instana/stanctl
fdisk /dev/nvme0n2  (options g,n,w)
mkfs.ext4 /dev/nvme0n2p1 
mount /dev/nvme0n2p1 /mnt/instana/stanctl 
cd /mnt/instana/stanctl
mkdir {data,metrics,analytics,objects} 
```

Verify config:

```shell
ls /mnt/instana/stanctl 
```

e.g. `analytics  data  lost+found  metrics  objects`

Add the following to `/etc/fstab`:

```shell
/dev/nvme0n2p1            /mnt/instana/stanctl       ext4    defaults        0 0
```

Set kernel parameters:

```shell
sh -c 'echo vm.swappiness=0 >> /etc/sysctl.d/99-stanctl.conf' && sysctl -p /etc/sysctl.d/99-stanctl.conf
sh -c 'echo fs.inotify.max_user_instances=8192 >> /etc/sysctl.d/99-stanctl.conf' && sysctl -p /etc/sysctl.d/99-stanctl.conf
grubby --args="transparent_hugepage=never" --update-kernel ALL
reboot
```

After the reboot, run:

```shell
cat /sys/kernel/mm/transparent_hugepage/enabled
```

The output should be:

```shell
always madvise [never]
```

Add `/usr/local/bin` to the PATH:

```shell
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
echo $PATH
```

If the path update is successful, you see `/usr/local/bin` in the output.

Check / set firewall:

```shell
systemctl status firewalld
```

If enabled, run:

```shell
firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=8443/tcp
firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16
firewall-cmd --permanent --zone=trusted --add-interface=lo
firewall-cmd --reload
```

## Bastion

On a the Bastion host.

Add repo and install `stanctl` command-line tool.

Adding Instana repo:

```shell
export DOWNLOAD_KEY=<Your Agent Key>

cat << EOF > /etc/yum.repos.d/Instana-Product.repo
[instana-product]
name=Instana-Product
baseurl=https://_:$DOWNLOAD_KEY@artifact-public.instana.io/artifactory/rel-rpm-public-virtual/
enabled=1
gpgcheck=0
gpgkey=https://_:$DOWNLOAD_KEY@artifact-public.instana.io/artifactory/api/security/keypair/public/repositories/rel-rpm-public-virtual
repo_gpgcheck=1
EOF
```

Then run:

```shell
yum clean expire-cache -y
yum update -y
yum install -y stanctl
stanctl air-gapped package
```

This command **may take a long time** to complete, in the end the following message will be displayed:

```shell
------------------------------------------
Air-gapped package successfully exported!
File: instana-airgapped.tar.gz
------------------------------------------
```

Copy the compressed file `instana-airgapped.tar.gz` to the Instana backend node.

## Instana Backend

Back to the Instana backend host.

Extract the air-gapped package, copy the stanctl file to the `/usr/local/bin` directory and run it:

```shell
tar -xzf /mnt/instana/instana-airgapped.tar.gz -C /usr/local/bin --strip-components 1 airgapped/stanctl
stanctl air-gapped import --file /mnt/instana/instana-airgapped.tar.gz
stanctl up --air-gapped
```

Expected output:

```shell
⠋ Adding Instana Helm repo  [0s] ✓
⠸ Storing the cluster binary  [0s] ✓
⠴ Setting up the cluster  [1s] ✓
⠼ Importing images [84/84]  [5m40s] ✓
⠋ Starting cluster components  [22s] ✓
⠇ Applying pre-requisites  [17s] ✓
⠋ Waiting for cert-manager  [0s] ✓
⠦ Applying data stores  [38s] ✓
⠙ Waiting for data stores  [3m18s] ✓
⠦ Applying backend apps  [1m23s] ✓
⠋ Starting backend  [0s] ✓
⠋ Waiting for Core/instana-core  [40s] ✓
⠋ Waiting for Unit/instana-unit  [40s] ✓
⠸ Unpinning old images  [0s] ✓

****************************************************************
* Successfully installed Instana Self-Hosted Standard Edition! *
*                                                              *
* URL: https://instana.localdomain                             *
* Username: admin@instana.local                                *
****************************************************************
```

## Troubleshooting

If `k3.service` fails to start, run:

```shell
ip route add default via <default gateway IP> 
```

Restart the last command if you see the error below:

```shell
⠏ Importing images [1/84]  [38s] ✕
Error: running "/usr/local/bin/ctr [-n k8s.io images import --label io.cri-containerd.pinned=pinned --label io.instana.pinned.timestamp=20250423113432 /root/.stanctl/airgapped/docker/images/agent_static---1.292.3.tar]" failed with exit code 1 (time="2025-04-23T11:34:32+10:00" level=warning msg="DEPRECATION: The `configs` property of `[plugins.\"io.containerd.grpc.v1.cri\".registry]` is deprecated since containerd v1.5 and will be removed in containerd v2.1.Use `config_path` instead."
time="2025-04-23T11:35:11+10:00" level=error msg="progress stream failed to recv" error="error reading from server: EOF"
time="2025-04-23T11:35:11+10:00" level=error msg="send stream ended without EOF" error="error reading from server: EOF"
time="2025-04-23T11:35:11+10:00" level=error msg="send failed" error=EOF
ctr: rpc error: code = Unavailable desc = error reading from server: EOF
)
```