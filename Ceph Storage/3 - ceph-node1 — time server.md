sudo timedatectl set-timezone Asia/Tehran

sudo apt install -y chrony

sudo systemctl disable --now systemd-timesyncd 2>/dev/null || true
sudo systemctl disable --now ntp 2>/dev/null || true
sudo systemctl disable --now ntpd 2>/dev/null || true

sudo cp -a /etc/chrony/chrony.conf \
    /etc/chrony/chrony.conf.bak.$(date +%Y%m%d-%H%M%S)

sudo tee /etc/chrony/chrony.conf > /dev/null <<'EOF'
local stratum 10
allow 192.168.56.0/24
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
echo "===== ENABLE CHRONY ====="
sudo systemctl enable chrony

echo
echo "===== RESTART CHRONY ====="
sudo systemctl restart chrony

sleep 3

echo
echo "===== SERVICE ====="
sudo systemctl status chrony --no-pager -l

echo
echo "===== TIME ====="
timedatectl

echo
echo "===== SOURCES ====="
chronyc sources -v

echo
echo "===== TRACKING ====="
chronyc tracking

echo
echo "===== NTP UDP 123 ====="
sudo ss -lunp | grep ':123' || true

echo
echo "===== NODE 1 IP ADDRESSES ====="
ip -br addr