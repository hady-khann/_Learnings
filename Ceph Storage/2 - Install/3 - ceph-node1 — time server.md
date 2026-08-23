sudo timedatectl set-timezone Asia/Tehran

sudo apt install -y chrony

sudo systemctl disable --now systemd-timesyncd 2>/dev/null || true

sudo tee /etc/chrony/chrony.conf > /dev/null <<'EOF'
local stratum 10
allow 192.168.0.0/16
driftfile /var/lib/chrony/chrony.drift
makestep 1.0 3
rtcsync
EOF

sudo chronyd -p -f /etc/chrony/chrony.conf

sudo systemctl enable --now chrony

sleep 5

echo "===== TIME ====="
timedatectl

echo
echo "===== CHRONY ====="
chronyc tracking

echo
echo "===== SOURCES ====="
chronyc sources -v

echo
echo "===== NTP PORT ====="
sudo ss -lunp | grep ':123'