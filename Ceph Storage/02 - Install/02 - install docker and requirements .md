# Install and Configure Required Packages and Docker

Run the following commands on each Ceph cluster node.

## 1. Update the System

```bash
sudo apt update -y
sudo apt upgrade -y
```

## 2. Install Required Packages

```bash
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  apt-transport-https \
  software-properties-common \
  chrony \
  lvm2 \
  python3 \
  python3-pip \
  openssh-server
```

## 3. Configure the Official Docker Repository

Create the Docker keyring directory:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download and install the Docker GPG key:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Set the appropriate permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Add the official Docker repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update the package index:

```bash
sudo apt update
```

## 4. Install Docker

Install Docker Engine and the required plugins:

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

## 5. Enable Required Services

Enable and start Docker:

```bash
sudo systemctl enable --now docker
```

Enable and start containerd:

```bash
sudo systemctl enable --now containerd
```

Enable and start Chrony:

```bash
sudo systemctl enable --now chrony
```

Enable and start SSH:

```bash
sudo systemctl enable --now ssh
```

## 6. Verify the Installation

### Docker

```bash
echo "===== DOCKER ====="

docker --version
docker compose version
```

### Containerd

```bash
echo
echo "===== CONTAINERD ====="

containerd --version
```

### Services

```bash
echo
echo "===== SERVICES ====="

systemctl is-active docker
systemctl is-active containerd
systemctl is-active chrony
systemctl is-active ssh
```

All four services should return:

```text
active
```

At this point, Docker, containerd, Chrony, and SSH are installed, enabled, and running.
