# Configure Hostnames in `/etc/hosts`

Add the hostnames and IP addresses of all Ceph cluster nodes to the `/etc/hosts` file.

On **ceph-node1**, edit the file:

```bash
sudo nano /etc/hosts
```

Add the following entries:

```text
################ Nodes Host Name ######################

192.168.111.100 ceph-controller
192.168.111.101 ceph-node1
192.168.111.102 ceph-node2
192.168.111.103 ceph-node3

########################################################
```

The complete `/etc/hosts` file should look similar to:

```text
127.0.0.1 localhost
127.0.1.1 ubuntu.localdomain

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

127.0.2.1 ceph-node1

################ Nodes Host Name ######################

192.168.111.100 ceph-controller
192.168.111.101 ceph-node1
192.168.111.102 ceph-node2
192.168.111.103 ceph-node3

########################################################
```

Repeat the hostname configuration on **ceph-node2** and **ceph-node3**, ensuring that all nodes have the same cluster hostname mappings.

Verify hostname resolution from each node:

```bash
getent hosts ceph-controller
getent hosts ceph-node1
getent hosts ceph-node2
getent hosts ceph-node3
```

All four hostnames should resolve to the expected `192.168.111.x` addresses.
