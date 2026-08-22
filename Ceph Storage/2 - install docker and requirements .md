sudo apt update -y
sudo apt upgrade -y

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

# Docker official repository
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# Enable required services
sudo systemctl enable --now docker
sudo systemctl enable --now containerd
sudo systemctl enable --now chrony
sudo systemctl enable --now ssh

# Verify
echo "===== DOCKER ====="
docker --version
docker compose version

echo
echo "===== CONTAINERD ====="
containerd --version

echo
echo "===== SERVICES ====="
systemctl is-active docker
systemctl is-active containerd
systemctl is-active chrony
systemctl is-active ssh