sudo timedatectl set-timezone Asia/Tehran

sudo apt install -y chrony

sudo systemctl disable --now systemd-timesyncd 2>/dev/null || true

sudo tee /etc/chrony/chrony.conf > /dev/null <<'EOF'
server ceph-node1 iburst
driftfile /var/lib/chrony/chrony.drift
makestep 1.0 3
rtcsync
EOF

sudo chronyd -p -f /etc/chrony/chrony.conf

sudo systemctl enable --now chrony

sleep 10

echo "===== TIME ====="
timedatectl

echo
echo "===== SOURCES ====="
chronyc sources -v

echo
echo "===== TRACKING ====="
chronyc tracking