# Install Cephadm and Bootstrap the Ceph Cluster

## Ceph Release Series

The following table lists the Ceph release series and their latest versions:

| Release series | Latest version |
| -------------- | -------------: |
| **Tentacle**   |         20.2.2 |
| **Squid**      |         19.2.5 |
| **Reef**       |         18.2.8 |
| **Quincy**     |         17.2.9 |
| **Pacific**    |        16.2.15 |
| **Octopus**    |        15.2.17 |
| **Nautilus**   |        14.2.22 |
| **Mimic**      |        13.2.10 |
| **Luminous**   |        12.2.13 |
| **Kraken**     |         11.2.1 |

> **Note:** Select the Ceph release according to your environment and compatibility requirements. For a new deployment, use a currently supported Ceph release rather than an end-of-life release.

---

## 1. Install Cephadm and Ceph Common

Update the package index:

```bash
sudo apt update
```

Install `cephadm` and `ceph-common`:

```bash
sudo apt install -y cephadm ceph-common
```

---

## 2. Verify the Installation

Check the installed `cephadm` version:

```bash
cephadm version
```

Check the Ceph CLI version:

```bash
ceph --version
```

Example:

```text
cephadm version <version>

ceph version <version>
```

> Run the version commands once each. There is no need to execute `cephadm version` twice.

---

## 3. Create an SSH Key for Ceph Cluster Management

Before bootstrapping and adding additional Ceph nodes, create an SSH key pair on the first Ceph node.

Cephadm uses SSH to connect to and manage the other nodes in the cluster. The SSH key generated here can be provided to `cephadm bootstrap`.

Generate an ED25519 SSH key:

```bash
ssh-keygen -t ed25519
```

When prompted for a passphrase, leave it empty by pressing **Enter** twice.

The generated files are:

```text
~/.ssh/ed25519
~/.ssh/ed25519.pub
```

Verify that the key files exist:

```bash
ls -l ~/.ssh/ed25519*
```

The public key will later be installed on the other Ceph nodes so that Cephadm can connect to them without requiring a password.

> **Important:** Keep the private key `~/.ssh/ed25519` secure. Never copy or distribute the private key to other Ceph nodes. Only the public key should be installed on remote hosts.

---

## 4. Bootstrap the Ceph Cluster

Bootstrap the cluster from the first Ceph node.

The monitor IP must be the IP address assigned to the first monitor node on the Ceph cluster network.

Use the SSH key created in the previous step:

```bash
sudo cephadm bootstrap --mon-ip <your-first-monitor-ip>
```

For example, if `ceph-node1` uses `192.168.111.101` as its monitor IP:

```bash
sudo cephadm bootstrap --mon-ip 192.168.111.101
```

The bootstrap process creates the initial Ceph cluster and deploys the first monitor and manager daemons.

After a successful bootstrap, verify the cluster:

```bash
sudo ceph -s
```

Also verify the orchestrator hosts:

```bash
sudo ceph orch host ls
```

The initial node should appear as the first host in the cluster.

> **Important:** Use the IP address reachable by the other Ceph nodes. Do not use `127.0.0.1` or a loopback address for `--mon-ip`.

> **Note:** If the SSH key is stored at the default path expected by your Cephadm version, explicitly specifying `--ssh-private-key` may not be necessary. Specifying it explicitly is recommended when using a dedicated Cephadm key such as `~/.ssh/ed25519`.

---

## 5. Prepare Node2 and Node3

Every host cephadm manages runs Ceph daemons as containers, so each new node needs a container runtime and time sync in place **before** it is added to the cluster.

On **node2** and **node3**, run:

```bash
sudo apt update
sudo apt install -y podman   # or docker, matching whatever node1 uses
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

Confirm the hostname on each node matches (or is resolvable to) what you intend to register with the orchestrator:

```bash
hostname
```

---

## 6. Distribute the Cluster SSH Key to Node2 and Node3

`cephadm bootstrap` generates its own cluster SSH keypair at `/etc/ceph/ceph.pub` / `/etc/ceph/ceph.key` on node1. The orchestrator uses this keypair — not any manually generated key — to connect to and manage every other host. Copy the **public** half to each new node's `root` account:

```bash
# From node1
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.111.102   # node2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.111.103   # node3
```

> **Tip:** You can confirm which public key the orchestrator is actually using with:
> ```bash
> sudo ceph cephadm get-pub-key
> ```

---

## 7. Add Node2 and Node3 to the Cluster

From node1, register each new host with the orchestrator:

```bash
sudo ceph orch host add ceph-node2 192.168.111.102
sudo ceph orch host add ceph-node3 192.168.111.103
```

The first argument must match the hostname reported by `hostname` on that node (or be resolvable via DNS/`/etc/hosts`).

Verify all three hosts are now recognized:

```bash
sudo ceph orch host ls
```

Expected output shows `ceph-node1`, `ceph-node2`, and `ceph-node3`, each reachable.

---

## 8. Expand the Monitor Quorum

A single monitor has no fault tolerance. With three hosts available, deploy monitors across all of them for proper quorum:

```bash
sudo ceph orch apply mon --placement="ceph-node1,ceph-node2,ceph-node3"
```

Check cluster health and monitor status:

```bash
sudo ceph -s
```

---

## 9. Discover Available Storage Devices

Before adding OSDs (the daemons that actually store data), list the raw, unused block devices cephadm can see across all hosts:

```bash
ceph orch device ls
```

This scans every host in the cluster and reports which disks are available for use — a device only qualifies if it's raw, unpartitioned, and has no existing filesystem or LVM data on it. Use this output to decide which devices to hand to `ceph orch apply osd` (or `ceph orch daemon add osd`) in the next step of OSD deployment.

---

## Summary of Commands (Nodes 2 & 3)

```bash
# On node2 and node3: prerequisites
sudo apt update
sudo apt install -y podman chrony
sudo systemctl enable --now chrony

# From node1: distribute SSH key
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.111.102   # node2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.111.103   # node3

# From node1: add hosts to the cluster
sudo ceph orch host add ceph-node2 192.168.111.102
sudo ceph orch host add ceph-node3 192.168.111.103
sudo ceph orch host ls

# From node1: expand monitor quorum
sudo ceph orch apply mon --placement="ceph-node1,ceph-node2,ceph-node3"

# From node1: discover storage devices across the cluster
ceph orch device ls
```
