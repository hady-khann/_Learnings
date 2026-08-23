# Configure Chrony as the Cluster Time Source

Configure Chrony on the **reference node** (`ceph-node1`) so it can provide time synchronization for the Ceph cluster without relying on an external NTP server.

## 1. Set the Timezone

Set the system timezone to `Asia/Tehran`:

```bash
sudo timedatectl set-timezone Asia/Tehran
```

## 2. Install Chrony

```bash
sudo apt install -y chrony
```

## 3. Disable `systemd-timesyncd`

Chrony should be the only time synchronization service managing NTP:

```bash
sudo systemctl disable --now systemd-timesyncd 2>/dev/null || true
```

## 4. Configure Chrony

Replace the existing Chrony configuration with the following:

```bash
sudo tee /etc/chrony/chrony.conf > /dev/null <<'EOF'
local stratum 10

allow 192.168.0.0/16

driftfile /var/lib/chrony/chrony.drift

makestep 1.0 3

rtcsync
EOF
```

### Configuration Notes

* `local stratum 10` allows this node to act as a local reference clock even without an upstream NTP server.
* `allow 192.168.0.0/16` allows hosts within the `192.168.0.0/16` network to synchronize with this node.
* `makestep 1.0 3` allows Chrony to immediately correct large time differences during the first three clock updates.
* `rtcsync` keeps the hardware clock synchronized with the system clock.
* `driftfile` stores the estimated clock drift.

## 5. Validate the Chrony Configuration

Before starting the service, verify that the configuration is valid:

```bash
sudo chronyd -p -f /etc/chrony/chrony.conf
```

The command should complete without configuration errors.

## 6. Enable and Start Chrony

```bash
sudo systemctl enable --now chrony
```

Allow a few seconds for Chrony to initialize:

```bash
sleep 5
```

## 7. Verify the Configuration

### System Time

```bash
echo "===== TIME ====="

timedatectl
```

Verify that the timezone is:

```text
Asia/Tehran
```

### Chrony Status

```bash
echo

echo "===== CHRONY ====="

chronyc tracking
```

### Chrony Sources

```bash
echo

echo "===== SOURCES ====="

chronyc sources -v
```

Because this node is configured as the local reference clock and does not use an external NTP server, it is expected that there may be no external NTP source.

### Verify NTP Port

Chrony should listen on UDP port `123`:

```bash
echo

echo "===== NTP PORT ====="

sudo ss -lunp | grep ':123'
```

You should see Chrony listening on UDP port `123`.

## 8. Important Cluster Configuration

This configuration is intended for **`ceph-node1` as the reference time source**.

The other Ceph nodes should synchronize their clocks from `ceph-node1` rather than using online NTP servers. Their Chrony configuration should therefore point to:

```text
server ceph-node1
```

or:

```text
server 192.168.111.101
```

Ensure that all Ceph nodes have synchronized clocks before proceeding with Ceph deployment.
