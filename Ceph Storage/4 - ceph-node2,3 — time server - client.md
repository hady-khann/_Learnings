sudo timedatectl set-timezone Asia/Tehran

sudo apt install -y chrony

sudo systemctl disable --now systemd-timesyncd 2>/dev/null || true
sudo systemctl disable --now ntp 2>/dev/null || true
sudo systemctl disable --now ntpd 2>/dev/null || true

sudo cp -a /etc/chrony/chrony.conf \
    /etc/chrony/chrony.conf.bak.$(date +%Y%m%d-%H%M%S)

sudo tee /etc/chrony/chrony.conf > /dev/null <<'EOF'
server ceph-node1 iburst prefer
driftfile /var/lib/chrony/chrony.drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF

echo "===== CHRONY CONFIG ====="
sudo cat /etc/chrony/chrony.conf

echo
echo "===== VALIDATE CONFIG ====="
sudo chronyd -p /etc/chrony/chrony.conf

echo
echo "===== RESOLVE NODE1 ====="
getent hosts ceph-node1

echo
echo "===== PING NODE1 ====="
ping -c 3 ceph-node1

echo
echo "===== ENABLE CHRONY ====="
sudo systemctl enable chrony

echo
echo "===== RESTART CHRONY ====="
sudo systemctl restart chrony

sleep 10

echo
echo "===== TIME ====="
timedatectl

echo
echo "===== CHRONY SERVICE ====="
sudo systemctl status chrony --no-pager -l

echo
echo "===== CHRONY SOURCES ====="
chronyc sources -v

echo
echo "===== CHRONY TRACKING ====="
chronyc tracking