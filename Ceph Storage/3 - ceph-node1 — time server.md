sudo timedatectl set-timezone Asia/Tehran

sudo apt install -y chrony

sudo systemctl disable --now systemd-timesyncd 2>/dev/null || true
sudo systemctl disable --now ntp 2>/dev/null || true
sudo systemctl disable --now ntpd 2>/dev/null || true

sudo cp -a /etc/chrony/chrony.conf \
    /etc/chrony/chrony.conf.bak.$(date +%Y%m%d-%H%M%S)

sudo tee /etc/chrony/chrony.conf > /dev/null <<'EOF'
# ============================================================
# ceph-node1
# INTERNAL CHRONY / NTP SERVER
# ============================================================

# Use this machine as the authoritative local time source.
# No Internet NTP servers are used.
local stratum 10

# Allow the Ceph/Vagrant network to synchronize from node1.
allow 192.168.56.0/24

# Clock drift.
driftfile /var/lib/chrony/chrony.drift

# Correct large initial offsets.
makestep 1.0 3

# Synchronize the hardware RTC with the system clock.
rtcsync

# Logging.
logdir /var/log/chrony
EOF

echo
echo "===== CHECK CHRONY CONFIG ====="
sudo chronyd -p /etc/chrony/chrony.conf

echo
echo "===== ENABLE CHRONY ====="
sudo systemctl enable chrony

echo
echo "===== RESTART CHRONY ====="
sudo systemctl restart chrony

sleep 3

echo
echo "============================================================"
echo " NODE 1 - TIME"
echo "============================================================"
timedatectl

echo
echo "============================================================"
echo " NODE 1 - CHRONY SERVICE"
echo "============================================================"
sudo systemctl status chrony --no-pager -l

echo
echo "============================================================"
echo " NODE 1 - CHRONY SOURCES"
echo "============================================================"
chronyc sources -v

echo
echo "============================================================"
echo " NODE 1 - CHRONY TRACKING"
echo "============================================================"
chronyc tracking

echo
echo "============================================================"
echo " NODE 1 - NTP PORT 123"
echo "============================================================"
sudo ss -lunp | grep ':123' || true