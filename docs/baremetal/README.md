# Sylva Bare-Metal Deployment Guide

This guide explains how to deploy Sylva on bare-metal infrastructure using Cluster API Provider Metal3, also known as CAPM3, then deploy O-DU and O-CU workloads on a Sylva-managed workload cluster.

Use this guide when the target environment is real physical servers instead of VMware VMs.

For the validated ProxmoxBMC lab path, including the HTTPS iPXE workaround and every problem/fix from the real deployment, start with [proxmoxbmc-validated-lab.md](proxmoxbmc-validated-lab.md). For the earlier Option B reference runbook, see [proxmoxbmc-option-b-runbook.md](proxmoxbmc-option-b-runbook.md).

## Bootstrap Options

The Bootstrap VM is the machine where you run the Sylva commands. It prepares the cluster, talks to the infrastructure provider, and starts the deployment.

For your Proxmox-based environment, these are the realistic options:

| Option | What It Means | Needs BMC? | Needs Nested Virtualization? | Best For |
| --- | --- | --- | --- | --- |
| Option A: Proxmox VM as bootstrap for real bare metal | Create an Ubuntu VM on Proxmox. It controls real physical servers through real BMC/IPMI/Redfish. | Yes, real BMC | No | Real CAPM3 bare-metal deployment |
| Option B: Bootstrap VM controls a separate Proxmox target VM | Create an Ubuntu bootstrap VM and a separate large target VM. Run ProxmoxBMC on the bootstrap VM. | Yes, emulated BMC through IPMI | No | Fake bare-metal lab without nesting VMs inside the bootstrap VM |
| Option C: Large Proxmox VM with nested fake BMC lab | Create one large Ubuntu VM. Inside it, run libvirt, fake bare-metal VMs, and VirtualBMC. | Yes, emulated BMC | Yes | Learning Metal3/CAPM3 when a Proxmox-aware emulator is not available |
| Option D: Large Proxmox VM with CAPD | Create one large Ubuntu VM. Run Docker-based Sylva CAPD lab inside it. | No | No | Easiest Sylva demo on Proxmox |
| Option E: Manual Kubernetes on Proxmox | Create VMs manually in Proxmox, install Kubernetes yourself, then deploy O-DU/O-CU. | No | No | O-RAN workload demo without full Sylva lifecycle |
| Option F: Nested VMware inside Proxmox | Run ESXi/vCenter inside Proxmox, then use CAPV. | No BMC, but needs vCenter | Yes | Only if the project must demonstrate VMware CAPV |

Recommended order:

1. Use **Option D** if the goal is the fastest working Sylva lab.
2. Use **Option B** if you want the bootstrap VM to emulate BMC control for a separate Proxmox target VM with ProxmoxBMC. Use the validated guide: [proxmoxbmc-validated-lab.md](proxmoxbmc-validated-lab.md).
3. Use **Option A** if the company gives you real physical servers with real BMC access.
4. Use **Option C** if you cannot get a Proxmox-aware emulator and can enable nested virtualization.
5. Use **Option E** if the goal is mainly O-DU/O-CU workload deployment, not Sylva cluster lifecycle.
6. Avoid **Option F** unless VMware CAPV is required, because nested vCenter/ESXi is heavy and fragile.

### Decision Diagram

```mermaid
flowchart TD
    start["What infrastructure do you have?"]
    realbm["Real physical servers with BMC?"]
    proxmox["Only Proxmox VMs?"]
    api["Can your BMC emulator control Proxmox API?"]
    enough["Can you enable nested virtualization?"]
    sylva["Need full Sylva platform demo?"]

    start --> realbm
    realbm -->|"Yes"| capm3["Use Option A<br/>Proxmox bootstrap VM + CAPM3 + real BMC"]
    realbm -->|"No"| proxmox
    proxmox --> api
    api -->|"Yes"| proxemu["Use Option B<br/>Bootstrap VM + ProxmoxBMC + target VM"]
    api -->|"No"| enough
    enough -->|"Yes"| fake["Use Option C<br/>Nested libvirt + VirtualBMC"]
    enough -->|"No"| sylva
    sylva -->|"Yes"| capd["Use Option D<br/>Large enough VM + Docker CAPD"]
    sylva -->|"No"| manual["Use Option E<br/>Manual Kubernetes + O-DU/O-CU"]
```

### My Recommendation for This Project

If your company only provides Proxmox and you do not have real BMC access yet, use this path first:

```text
Proxmox
|
|-- Large Ubuntu VM
    |
    |-- Docker
    |-- Sylva CAPD lab
    |-- O-DU/O-CU workload demo
```

Then use the Proxmox-aware fake BMC path as a second milestone:

```text
Proxmox
|
|-- Bootstrap VM
|     |
|     |-- ProxmoxBMC
|     |-- Metal3/CAPM3 learning lab
|
|-- Large Target VM
      |
      |-- treated as fake bare metal
```

Use the real CAPM3 path only when you have:

- Physical servers
- BMC/IPMI/Redfish access
- PXE/provisioning network
- Management network
- Enough nodes for the management and workload clusters

## Bare-Metal Workflow

```mermaid
flowchart LR
    a["Prepare bootstrap VM"] --> b["Prepare bare-metal networks"]
    b --> c["Collect server inventory and BMC access"]
    c --> d["Prepare PXE / DHCP / provisioning path"]
    d --> e["Clone sylva-core"]
    e --> f["Copy CAPM3 environment values"]
    f --> g["Configure values.yaml and secrets.yaml"]
    g --> h["Validate YAML"]
    h --> i["Deploy Sylva management cluster"]
    i --> j["Verify Metal3 and Cluster API"]
    j --> k["Create workload cluster"]
    k --> l["Deploy O-CU"]
    l --> m["Deploy O-DU"]
    m --> n["Validate O-RAN workload"]
```

## Bare-Metal Architecture

```mermaid
flowchart TB
    operator["Admin / Operator"]
    bootstrap["Bootstrap VM<br/>Ubuntu 22.04"]
    repo["sylva-core<br/>CAPM3 environment values"]

    subgraph networks["Bare-Metal Networks"]
        mgmt_net["Management network"]
        prov_net["Provisioning / PXE network"]
        bmc_net["BMC network<br/>IPMI / Redfish"]
        workload_net["Workload / RAN network"]
    end

    subgraph metal3["Metal3 Provisioning Stack"]
        ironic["Ironic"]
        ipa["Ironic Python Agent"]
        dhcp["DHCP / PXE / HTTP boot services"]
    end

    subgraph servers["Physical Servers"]
        bmc1["BMC 1"]
        bmc2["BMC 2"]
        bmc3["BMC 3"]
        node1["Management node 1"]
        node2["Management node 2"]
        node3["Management node 3"]
        worker1["Workload node 1"]
        worker2["Workload node 2"]
    end

    subgraph sylva["Sylva Management Cluster"]
        rke2["RKE2 Kubernetes"]
        rancher["Rancher"]
        capi["Cluster API"]
        capm3["CAPM3 Controller"]
        metal3_ctrl["Metal3 Controller"]
        flux["FluxCD"]
        harbor["Harbor"]
        vault["Vault"]
        keycloak["Keycloak"]
        monitoring["Monitoring"]
    end

    subgraph workload["Sylva Workload Cluster"]
        ns["Namespace: oran"]
        ocu["O-CU / CU Stub"]
        odu["O-DU High"]
        ran_net["Optional SR-IOV / DPDK / PTP"]
    end

    operator --> bootstrap
    bootstrap --> repo
    bootstrap --> mgmt_net
    bootstrap --> prov_net
    bootstrap --> bmc_net
    capi --> capm3
    capm3 --> metal3_ctrl
    metal3_ctrl --> ironic
    ironic --> dhcp
    dhcp --> prov_net
    bmc_net --> bmc1
    bmc_net --> bmc2
    bmc_net --> bmc3
    bmc1 --> node1
    bmc2 --> node2
    bmc3 --> node3
    prov_net --> node1
    prov_net --> node2
    prov_net --> node3
    node1 --> sylva
    node2 --> sylva
    node3 --> sylva
    rancher --> workload
    flux --> workload
    harbor --> ocu
    harbor --> odu
    ns --> ocu
    ns --> odu
    workload_net --> ran_net
    odu -->|"F1 / SCTP"| ocu
```

## What You Manually Create

You manually prepare:

- Bootstrap VM
- Physical servers
- BMC access for each server
- Management network
- Provisioning/PXE network
- Optional workload/RAN network
- DHCP/PXE design
- Server inventory

Sylva, Cluster API, CAPM3, and Metal3 then provision the management and workload cluster nodes.

## Requirements

### Bootstrap VM

| Component | Recommended |
| --- | --- |
| OS | Ubuntu 22.04 LTS |
| CPU | 4+ vCPU |
| RAM | 16 GB |
| Disk | 50+ GB |
| Access | SSH and passwordless sudo |
| Network | Access to management, provisioning, and BMC networks |

### Bare-Metal Servers

Minimum management cluster:

- 3 physical servers for management control plane
- 4+ CPU cores per server
- 16 GB RAM per server
- 100 GB disk per server
- One or more NICs per server
- BMC access through IPMI or Redfish

Recommended lab:

- 3 management servers
- 2 or more workload servers
- 8+ CPU cores per server
- 32+ GB RAM per server
- 250+ GB disk per server
- Separate NICs or VLANs for provisioning, management, and workload traffic

### Network Requirements

| Network | Purpose |
| --- | --- |
| Management network | Kubernetes API, node management, Rancher, Sylva services |
| Provisioning network | PXE boot, DHCP, Ironic provisioning, image download |
| BMC network | IPMI or Redfish access to power-cycle and provision servers |
| Workload/RAN network | O-DU/O-CU traffic and optional RAN interfaces |

You need:

- DHCP or static IP plan for bare-metal nodes
- Free virtual IP for Kubernetes API or ingress, depending on your Sylva configuration
- DNS entries or `/etc/hosts` records for Sylva services
- NTP reachable from every node
- Internet or registry mirror access for images
- Firewall rules allowing bootstrap VM to reach BMCs and provisioning services

### Server Inventory

Collect this information before deployment:

| Field | Example |
| --- | --- |
| Server name | `bm-mgmt-01` |
| BMC address | `192.168.100.11` |
| BMC type | `ipmi` or `redfish` |
| BMC username | `admin` |
| BMC password | Stored in `secrets.yaml` |
| Boot MAC address | `52:54:00:aa:bb:cc` |
| Provisioning NIC | `eno1` |
| Management IP | `192.168.10.21` |
| Role | Management or workload |

## Step 1: Prepare the Bootstrap VM

Run all Sylva commands from the Bootstrap VM.

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
    curl \
    wget \
    git \
    vim \
    jq \
    unzip \
    tar \
    make \
    python3 \
    python3-pip \
    ca-certificates \
    gnupg \
    lsb-release \
    yamllint
```

Install Docker:

```bash
sudo apt remove docker docker-engine docker.io containerd runc -y
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) \
signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

Install `kubectl`, `helm`, `clusterctl`, and `yq`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

curl -L https://github.com/kubernetes-sigs/cluster-api/releases/latest/download/clusterctl-linux-amd64 \
-o clusterctl
chmod +x clusterctl
sudo mv clusterctl /usr/local/bin/

sudo wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 \
-O /usr/local/bin/yq
sudo chmod +x /usr/local/bin/yq
```

Verify:

```bash
docker version
kubectl version --client
helm version
clusterctl version
yq --version
yamllint --version
```

## Step 2: Prepare Bare-Metal Networking

Create or confirm these networks:

```text
Management network:
  Example: 192.168.10.0/24
  Used by Kubernetes nodes and Sylva services

Provisioning network:
  Example: 192.168.20.0/24
  Used by PXE, DHCP, Ironic, and OS image provisioning

BMC network:
  Example: 192.168.100.0/24
  Used for IPMI or Redfish access

Workload/RAN network:
  Example: 192.168.30.0/24
  Used by O-DU/O-CU workloads and optional RAN traffic
```

From the Bootstrap VM, test reachability:

```bash
ping <bmc-ip>
ping <management-gateway>
ping <provisioning-gateway>
```

## Step 3: Prepare BMC Access

Check that the Bootstrap VM can control server power through BMC.

For IPMI:

```bash
sudo apt install -y ipmitool
ipmitool -I lanplus -H <bmc-ip> -U <bmc-user> -P '<bmc-password>' power status
```

For Redfish, test with curl:

```bash
curl -k -u <bmc-user>:<bmc-password> https://<bmc-ip>/redfish/v1/
```

Do this for every physical server.

## Step 4: Optional ProxmoxBMC Fake BMC Lab

Use this section if you want the Bootstrap VM to act as a fake BMC and control a separate target VM on Proxmox.

This option uses **ProxmoxBMC**, a third-party tool based on VirtualBMC. The important difference is that ProxmoxBMC talks to the **Proxmox API** instead of `libvirt`.

This design does not require nested virtualization because the fake bare-metal target is a normal Proxmox VM, separate from the Bootstrap VM.

```text
Proxmox
|
|-- Bootstrap VM
|     Ubuntu 22.04
|     Runs Sylva / Metal3 / CAPM3 tools
|     Runs ProxmoxBMC
|
|-- Target VM
      VMID 200
      Treated as fake bare metal
      Controlled by ProxmoxBMC through Proxmox API
      PXE/network boots through net0
```

### ProxmoxBMC Architecture

```mermaid
flowchart TB
    proxmox["Proxmox Server"]
    api["Proxmox API"]

    subgraph bootstrap["Bootstrap VM"]
        sylva["Sylva / Metal3 / CAPM3"]
        pbmcd["pbmcd daemon"]
        pbmc["pbmc CLI"]
    end

    subgraph target["Target VM"]
        fakebm["fake-bm-01<br/>VMID 200"]
        net0["net0<br/>PXE / provisioning network"]
        disk["scsi0<br/>Provisioned disk"]
    end

    sylva -->|"IPMI<br/>ipmi://bootstrap-ip:6625"| pbmcd
    pbmc --> pbmcd
    pbmcd -->|"Proxmox API token"| api
    api --> proxmox
    proxmox --> target
    sylva -->|"PXE / iPXE provisioning"| net0
    net0 --> disk
```

### 4.1 Important Validation Notes

This is the validated behavior for this option:

- ProxmoxBMC exposes an IPMI endpoint.
- ProxmoxBMC controls a Proxmox VM by VMID.
- The target VM must be separate from the Bootstrap VM.
- Metal3 should use an `ipmi://` BMC address.
- For IPMI, Metal3 uses network boot/iPXE, so the provisioning network and DHCP path must work.
- ProxmoxBMC assumes:
  - `network` boot means `net0`
  - `hd` boot means `scsi0`
  - `cdrom` boot means `ide2`

Because of that, configure the target VM with:

```text
Boot NIC: net0
Main disk: scsi0
Boot order: net0 first for the lab
```

### 4.2 Create the Bootstrap VM

Create an Ubuntu 22.04 VM on Proxmox.

Recommended sizing:

| Component | Value |
| --- | --- |
| CPU | 4+ vCPU |
| RAM | 8 to 16 GB |
| Disk | 50+ GB |
| Network 1 | Management network |
| Network 2 | Provisioning network, if separated |
| Access | SSH and passwordless sudo |

Install basic tools:

```bash
sudo apt update
sudo apt install -y \
    curl \
    jq \
    git \
    python3 \
    python3-pip \
    ipmitool \
    python3-venv \
    ca-certificates
```

### 4.3 Create the Target VM in Proxmox

Create a separate Proxmox VM that Metal3 will treat as fake bare metal.

Example target VM:

| Component | Value |
| --- | --- |
| Name | `fake-bm-01` |
| VM ID | `200` |
| CPU | 4+ vCPU |
| RAM | 8+ GB |
| Disk | 50+ GB |
| Disk bus | `scsi0` |
| NIC | `net0`, connected to provisioning/management bridge |
| Boot order | `net0` first, then `scsi0` |
| OS install | No manual OS install; Metal3 should provision it |

Record:

```text
Target VM ID: 200
Target VM name: fake-bm-01
Target boot MAC: <mac-address>
Proxmox node name: <proxmox-node-name>
```

The boot MAC is required later by Metal3:

```yaml
bootMACAddress: "<mac-address>"
```

### 4.4 Create a Proxmox API Token

In the Proxmox UI:

1. Go to **Datacenter > Permissions > API Tokens**.
2. Create a token for a user that can manage the target VM.
3. Give the token permissions only for the target VM where possible.

The emulator needs permissions similar to:

- `VM.Audit`
- `VM.PowerMgmt`
- `VM.Config.Options`

Save:

```text
PVE_HOST=<proxmox-hostname-or-ip>
PVE_NODE=<proxmox-node-name>
PVE_TOKEN_USER=<user@realm>
PVE_TOKEN_NAME=<token-name>
PVE_TOKEN_ID=<user@realm!token-name>
PVE_TOKEN_SECRET=<token-secret>
TARGET_VMID=200
```

Do not commit the token secret to Git.

### 4.5 Test Proxmox API Access from the Bootstrap VM

Set variables on the Bootstrap VM:

```bash
export PVE_HOST="<proxmox-hostname-or-ip>"
export PVE_NODE="<proxmox-node-name>"
export PVE_TOKEN_USER="<user@realm>"
export PVE_TOKEN_NAME="<token-name>"
export PVE_TOKEN_ID="<user@realm!token-name>"
export PVE_TOKEN_SECRET="<token-secret>"
export TARGET_VMID="200"
```

`PVE_NODE` is only needed for these direct `curl` tests. ProxmoxBMC itself takes `--proxmox-address` and the VMID.

Check target VM status:

```bash
curl -k \
  -H "Authorization: PVEAPIToken=${PVE_TOKEN_ID}=${PVE_TOKEN_SECRET}" \
  "https://${PVE_HOST}:8006/api2/json/nodes/${PVE_NODE}/qemu/${TARGET_VMID}/status/current" | jq
```

Power on:

```bash
curl -k -X POST \
  -H "Authorization: PVEAPIToken=${PVE_TOKEN_ID}=${PVE_TOKEN_SECRET}" \
  "https://${PVE_HOST}:8006/api2/json/nodes/${PVE_NODE}/qemu/${TARGET_VMID}/status/start" | jq
```

Power off:

```bash
curl -k -X POST \
  -H "Authorization: PVEAPIToken=${PVE_TOKEN_ID}=${PVE_TOKEN_SECRET}" \
  "https://${PVE_HOST}:8006/api2/json/nodes/${PVE_NODE}/qemu/${TARGET_VMID}/status/stop" | jq
```

If these API calls do not work, the BMC emulator will not be able to control the target VM.

### 4.6 Install ProxmoxBMC on the Bootstrap VM

Run on the Bootstrap VM:

```bash
cd ~
git clone https://github.com/agnon/proxmoxbmc.git
cd proxmoxbmc

python3 -m venv .env
. .env/bin/activate
pip install -r requirements.txt
pip install .
```

Verify the commands exist inside the virtual environment:

```bash
pbmc --help
pbmcd --help
```

### 4.7 Start the ProxmoxBMC Daemon

For the first test, keep the daemon running in one terminal:

```bash
cd ~/proxmoxbmc
. .env/bin/activate
pbmcd
```

Open a second terminal on the Bootstrap VM for the remaining commands.

### 4.8 Add the Target VM to ProxmoxBMC

Use a high UDP port such as `6625`. Port `623` is the standard IPMI port, but binding to it may require root.

In the second terminal:

```bash
cd ~/proxmoxbmc
. .env/bin/activate

pbmc add \
  --username admin \
  --password password \
  --port 6625 \
  --address 0.0.0.0 \
  --proxmox-address "${PVE_HOST}" \
  --token-user "${PVE_TOKEN_USER}" \
  --token-name "${PVE_TOKEN_NAME}" \
  --token-value "${PVE_TOKEN_SECRET}" \
  "${TARGET_VMID}"
```

Example:

```bash
pbmc add \
  --username admin \
  --password password \
  --port 6625 \
  --address 0.0.0.0 \
  --proxmox-address 192.168.1.50 \
  --token-user root@pam \
  --token-name bmc \
  --token-value "<token-secret>" \
  200
```

Start the fake BMC:

```bash
pbmc start "${TARGET_VMID}"
pbmc list
```

### 4.9 Test IPMI Against ProxmoxBMC

Test locally from the Bootstrap VM:

```bash
ipmitool -I lanplus -H 127.0.0.1 -p 6625 -U admin -P password power status
ipmitool -I lanplus -H 127.0.0.1 -p 6625 -U admin -P password power on
ipmitool -I lanplus -H 127.0.0.1 -p 6625 -U admin -P password power off
```

Then test using the Bootstrap VM IP. This is closer to how Metal3 will reach it:

```bash
ipmitool -I lanplus -H <bootstrap-vm-ip> -p 6625 -U admin -P password power status
```

If a firewall is enabled on the Bootstrap VM, allow the IPMI UDP port:

```bash
sudo ufw allow 6625/udp
```

### 4.10 Run ProxmoxBMC as a systemd Service

After the manual test works, create a service:

```bash
sudo vim /etc/systemd/system/pbmcd.service
```

Example:

```ini
[Unit]
Description=pbmcd service
After=network.target

[Service]
ExecStart=/home/ubuntu/proxmoxbmc/.env/bin/pbmcd --foreground
Restart=on-failure
RestartSec=2
TimeoutSec=120
Type=simple
User=ubuntu
Group=ubuntu

[Install]
WantedBy=multi-user.target
```

Adjust `/home/ubuntu/proxmoxbmc/.env/bin/pbmcd` and the user/group to match your Bootstrap VM.

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable pbmcd
sudo systemctl start pbmcd
sudo systemctl status pbmcd
```

### 4.11 Use the Fake BMC in Metal3 or Sylva

Metal3 should use the Bootstrap VM IP and the ProxmoxBMC port:

```text
ipmi://<bootstrap-vm-ip>:6625
```

Example BareMetalHost:

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: fake-bm-01
spec:
  online: true
  bootMACAddress: "<target-vm-mac-address>"
  bmc:
    address: "ipmi://<bootstrap-vm-ip>:6625"
    credentialsName: fake-bm-01-bmc-secret
```

Example secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: fake-bm-01-bmc-secret
type: Opaque
stringData:
  username: admin
  password: password
```

### 4.12 Validation Checklist

Before using this with Sylva/CAPM3, confirm:

- `pbmcd` is running.
- `pbmc list` shows the target VM.
- `ipmitool` can power the target VM on and off.
- The target VM NIC used for provisioning is `net0`.
- The target VM main disk is `scsi0`.
- The target VM can PXE/network boot.
- Metal3 can reach `ipmi://<bootstrap-vm-ip>:6625`.
- The provisioning network has DHCP/iPXE access.

### 4.13 Important Limitations

Important limits:

- ProxmoxBMC is a third-party lab tool, not an official Sylva or Metal3 provider.
- ProxmoxBMC exposes IPMI, not Redfish.
- Plain `VirtualBMC` still does not control Proxmox VMs directly.
- The target VM must be separate from the Bootstrap VM.
- The target VM must use `net0` for network boot to match ProxmoxBMC assumptions.
- The target VM should use `scsi0` for disk boot to match ProxmoxBMC assumptions.
- One fake BMC is useful for learning, but not enough for a full HA Sylva management cluster.

For a full fake bare-metal Sylva lab, repeat the same pattern:

```text
fake-bm-01 -> Proxmox VM 200 -> ipmi://bootstrap-ip:6625
fake-bm-02 -> Proxmox VM 201 -> ipmi://bootstrap-ip:6626
fake-bm-03 -> Proxmox VM 202 -> ipmi://bootstrap-ip:6627
```

### 4.14 Alternative: Nested libvirt + VirtualBMC

If ProxmoxBMC does not work in your environment, use the nested libvirt method instead:

```text
Proxmox
|
|-- Large Ubuntu VM
    |
    |-- libvirt/KVM
    |-- VirtualBMC
    |-- fake-bm-01 VM
```

That alternative requires nested virtualization, but it works with the standard `VirtualBMC` tool.

## Step 5: Configure Server Boot Order

In each server BIOS or BMC console:

1. Enable network boot.
2. Put the provisioning NIC first in the boot order.
3. Enable UEFI or legacy boot consistently across all servers.
4. Disable Secure Boot unless your image and boot chain support it.
5. Confirm the boot MAC address for each server.

The boot MAC address is required by Metal3 so it can match each physical or fake bare-metal host to the correct provisioning interface.

## Step 6: Clone Sylva Repository

```bash
git clone https://gitlab.com/sylva-projects/sylva-core.git
cd sylva-core
```

For repeatable deployments, pin a tested Sylva release:

```bash
git tag --list
git checkout <tested-sylva-release>
```

## Step 7: Create the CAPM3 Environment Folder

Copy the bare-metal CAPM3 sample:

```bash
cp -r environment-values/rke2-capm3/ environment-values/my-rke2-capm3
cd environment-values/my-rke2-capm3
```

If the exact folder name is different in your Sylva release, inspect:

```bash
ls ../../environment-values
```

## Step 8: Configure values.yaml

Edit:

```bash
vim values.yaml
```

Set the management cluster name, Kubernetes version, network settings, Metal3 settings, and enabled Sylva units.

Example structure:

```yaml
name: sylva-baremetal-management

k8s_version: v1.30.1+rke2r1

control_plane_replicas: 3

capi_providers:
  infra_provider: capm3
  bootstrap_provider: cabpr

cluster_virtual_ip: "192.168.10.50"

ntp:
  servers:
    - "0.pool.ntp.org"
    - "1.pool.ntp.org"

units:
  rancher:
    enabled: true
  keycloak:
    enabled: true
  harbor:
    enabled: true
  vault:
    enabled: true
  longhorn:
    enabled: true
```

Add the CAPM3 or Metal3 fields required by your selected Sylva release. These normally include:

- Provisioning network
- Management network
- Bare-metal host inventory
- Boot MAC addresses
- BMC addresses
- Node image URL or image reference
- API virtual IP
- Ingress virtual IP

## Step 9: Configure secrets.yaml

Edit:

```bash
vim secrets.yaml
```

Store BMC credentials and sensitive values here.

Example structure:

```yaml
baremetal_hosts:
  bm-mgmt-01:
    bmc:
      address: "ipmi://192.168.100.11"
      username: "admin"
      password: "YourSecurePassword"
  bm-mgmt-02:
    bmc:
      address: "ipmi://192.168.100.12"
      username: "admin"
      password: "YourSecurePassword"
  bm-mgmt-03:
    bmc:
      address: "ipmi://192.168.100.13"
      username: "admin"
      password: "YourSecurePassword"
```

Do not commit real `secrets.yaml` values to Git.

## Step 10: Validate YAML

```bash
yamllint values.yaml
yamllint secrets.yaml
```

Return to the Sylva repository root:

```bash
cd ../../
```

## Step 11: Deploy Sylva Management Cluster

Run the deployment workflow supported by your Sylva release.

Common pattern:

```bash
make all ENV=environment-values/my-rke2-capm3
```

If your selected Sylva release uses a bootstrap script instead:

```bash
./bootstrap.sh environment-values/my-rke2-capm3
```

The deployment should:

- Start a bootstrap cluster
- Install Cluster API
- Install CAPM3 and Metal3
- Register bare-metal hosts
- Provision management cluster nodes
- Install RKE2
- Install Sylva platform units

## Step 12: Monitor Bare-Metal Provisioning

Watch Cluster API objects:

```bash
kubectl get clusters -A
kubectl get machines -A
kubectl get machinedeployments -A
```

Watch Metal3 objects:

```bash
kubectl get baremetalhosts -A
kubectl get metal3machines -A
```

Watch pods:

```bash
kubectl get pods -A -w
```

Check Metal3 and CAPM3 logs:

```bash
kubectl logs -n capm3-system deployment/capm3-controller-manager
kubectl logs -n baremetal-operator-system deployment/baremetal-operator-controller-manager
```

## Step 13: Verify Management Cluster

Get the management cluster kubeconfig using the method for your Sylva release, then verify:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
kubectl get sylvaunits -A
kubectl get gitrepositories -A
```

Access Rancher:

```bash
kubectl get ingress -A
```

## Step 14: Create or Select the Workload Cluster

O-DU and O-CU should run on a workload cluster, not directly on the management cluster.

If a workload cluster must be created, inspect the workload cluster examples:

```bash
ls environment-values/workload-clusters
```

Copy the CAPM3 workload sample:

```bash
cp -r environment-values/workload-clusters/<capm3-workload-sample> environment-values/workload-clusters/oran-baremetal-workload
```

Edit:

```bash
vim environment-values/workload-clusters/oran-baremetal-workload/values.yaml
vim environment-values/workload-clusters/oran-baremetal-workload/secrets.yaml
```

Deploy:

```bash
make workload-cluster ENV=environment-values/workload-clusters/oran-baremetal-workload
```

If your release provides a script:

```bash
./apply-workload-cluster.sh environment-values/workload-clusters/oran-baremetal-workload
```

Validate:

```bash
kubectl get clusters -A
kubectl get machines -A
kubectl get baremetalhosts -A
```

## Step 15: Prepare O-DU and O-CU Workload Nodes

For a simple lab, Kubernetes service networking may be enough.

For a realistic RAN lab, prepare the workload nodes for:

- Multus
- SR-IOV
- Hugepages
- CPU pinning
- DPDK
- PTP
- Real-time kernel tuning
- Dedicated NICs for RAN traffic

Confirm node labels:

```bash
kubectl get nodes --show-labels
```

Example labels:

```bash
kubectl label node <node-name> node-role.oran/o-du=true
kubectl label node <node-name> node-role.oran/o-cu=true
```

## Step 16: Deploy O-CU

Create the namespace:

```bash
kubectl create namespace oran
```

Deploy O-CU or a CU stub first:

```bash
kubectl apply -f o-cu-deployment.yaml
kubectl apply -f o-cu-service.yaml
```

The O-CU service should expose the F1 endpoint:

```bash
kubectl get svc -n oran
```

## Step 17: Deploy O-DU

Deploy O-DU after O-CU is running:

```bash
kubectl apply -f o-du-deployment.yaml
```

The O-DU should point to:

```text
o-cu.oran.svc.cluster.local
```

For advanced bare-metal RAN testing, add host networking, SR-IOV interfaces, hugepages, DPDK, and PTP only after the base deployment works.

## Step 18: Verify O-RAN Workloads

```bash
kubectl get all -n oran
kubectl get pods -n oran -o wide
kubectl logs -n oran deploy/o-cu
kubectl logs -n oran deploy/o-du
kubectl describe svc o-cu -n oran
kubectl top pods -n oran
```

Success criteria:

- Management cluster is healthy.
- Workload cluster is healthy.
- Bare-metal hosts are provisioned.
- Rancher sees the clusters.
- O-CU is running.
- O-DU is running.
- O-DU reaches O-CU through the F1 service.
- Logs and monitoring show stable workload behavior.

## Troubleshooting

### Bare-Metal Host Does Not Power On

Check:

- BMC IP address
- BMC username and password
- IPMI or Redfish protocol
- Firewall access from Bootstrap VM to BMC network

Commands:

```bash
ipmitool -I lanplus -H <bmc-ip> -U <bmc-user> -P '<bmc-password>' power status
kubectl describe baremetalhost <host-name> -n <namespace>
```

### Server Does Not PXE Boot

Check:

- PXE enabled in BIOS
- Correct provisioning NIC
- Correct boot MAC address
- DHCP scope
- Provisioning VLAN
- Ironic and DHCP pods

Commands:

```bash
kubectl get pods -A | grep -E "ironic|metal3|dhcp"
kubectl describe baremetalhost <host-name> -n <namespace>
```

### Node Provisioning Fails

Check:

- OS image URL or registry access
- Disk availability
- Network access from node to image server
- Metal3 logs
- BareMetalHost status

Commands:

```bash
kubectl get baremetalhosts -A
kubectl describe baremetalhost <host-name> -n <namespace>
kubectl logs -n baremetal-operator-system deployment/baremetal-operator-controller-manager
```

### O-DU or O-CU Fails

Check:

- Image pull credentials
- SCTP support
- ConfigMaps
- Secrets
- Service name
- Node labels
- SR-IOV and Multus configuration
- Hugepages
- CPU and memory requests

## Cleanup

Delete the workload cluster first:

```bash
kubectl delete cluster <workload-cluster-name>
```

Delete the management cluster:

```bash
kubectl delete cluster sylva-baremetal-management
```

Power-cycle or reset bare-metal servers only after Cluster API and Metal3 objects are cleaned up.

## References

- Sylva core repository: https://gitlab.com/sylva-projects/sylva-core
- Sylva documentation: https://sylva-projects.gitlab.io/docs/
- Cluster API: https://cluster-api.sigs.k8s.io/
- Metal3: https://metal3.io/
- Metal3 supported BMC protocols: https://book.metal3.io/bmo/supported_hardware
- ProxmoxBMC: https://github.com/agnon/proxmoxbmc
- O-RAN SC documentation: https://docs.o-ran-sc.org/
