sudo timedatectl set-timezone Asia/Tehran

sudo apt install -y chrony

sudo systemctl disable --now systemd-timesyncd 2>/dev/null || true
sudo systemctl disable --now ntp 2>/dev/null || true
sudo systemctl disable --now ntpd 2>/dev/null || true

sudo cp -a /etc/chrony/chrony.conf \
    /etc/chrony/chrony.conf.bak.$(date +%Y%m%d-%H%M%S)

sudo tee /etc/chrony/chrony.conf > /dev/null <<'EOF'
# ============================================================
# Ceph node - INTERNAL CHRONY / NTP CLIENT
# ============================================================

# ceph-node1 is the internal NTP server.
server ceph-node1 iburst prefer

# Clock drift.
driftfile /var/lib/chrony/chrony.drift

# Correct large initial offsets.
makestep 1.0 3

# Synchronize hardware RTC with system clock.
rtcsync

# Logging.
logdir /var/log/chrony
EOF

echo "===== CHECK CHRONY CONFIG ====="
sudo chronyd -p /etc/chrony/chrony.conf

echo
echo "===== ENABLE CHRONY ====="
sudo systemctl enable chrony

echo
echo "===== RESTART CHRONY ====="
sudo systemctl restart chrony

sleep 5

echo
echo "============================================================"
echo " TIME"
echo "============================================================"
timedatectl

echo
echo "============================================================"
echo " CHRONY SERVICE"
echo "============================================================"
sudo systemctl status chrony --no-pager -l

echo
echo "============================================================"
echo " CHRONY SOURCES"
echo "============================================================"
chronyc sources -v

echo
echo "============================================================"
echo " CHRONY TRACKING"
echo "============================================================"
chronyc tracking