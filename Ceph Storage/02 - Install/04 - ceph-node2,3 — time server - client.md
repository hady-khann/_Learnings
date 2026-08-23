# Configure Chrony Client Nodes

Configure Chrony on the **Ceph client nodes** (`ceph-node2` and `ceph-node3`) so they synchronize their system clocks from the reference node `ceph-node1`.

No external or online NTP server is required.

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
server ceph-node1 iburst

driftfile /var/lib/chrony/chrony.drift

makestep 1.0 3

rtcsync
EOF
```

### Configuration Notes

* `server ceph-node1 iburst` configures `ceph-node1` as the NTP source.
* `iburst` accelerates the initial synchronization process.
* `driftfile` stores the estimated clock drift.
* `makestep 1.0 3` allows Chrony to make a large time correction during the first three clock updates.
* `rtcsync` keeps the hardware clock synchronized with the system clock.

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

Allow Chrony time to establish synchronization with `ceph-node1`:

```bash
sleep 10
```

## 7. Verify the Time

```bash
echo "===== TIME ====="

timedatectl
```

Verify that the timezone is:

```text
Asia/Tehran
```

## 8. Verify the NTP Source

```bash
echo

echo "===== SOURCES ====="

chronyc sources -v
```

The output should show `ceph-node1` as the configured time source.

A healthy synchronized source will typically be marked with `^*` in the `chronyc sources -v` output.

## 9. Verify Synchronization Tracking

```bash
echo

echo "===== TRACKING ====="

chronyc tracking
```

Check that Chrony reports a valid reference and that the system is synchronized.

## 10. Repeat on Other Ceph Nodes

Run this configuration on:

```text
ceph-node2
ceph-node3
```

Both nodes should use:

```text
ceph-node1
```

as their **only NTP source**.

The intended topology is:

```text
                    ┌─────────────────┐
                    │   ceph-node1    │
                    │ Reference NTP   │
                    │  Local Stratum  │
                    └────────┬────────┘
                             │
                  ┌──────────┴──────────┐
                  │                     │
          ┌───────▼────────┐   ┌────────▼───────┐
          │   ceph-node2   │   │   ceph-node3   │
          │  Chrony Client │   │  Chrony Client │
          └────────────────┘   └────────────────┘
```

This ensures that all Ceph nodes use the same internal time source and do not depend on an online NTP service.
