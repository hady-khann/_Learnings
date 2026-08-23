write host names in hosts file 


vagrant@ceph-node1:~$ cat /etc/hosts 
127.0.0.1 localhost
127.0.1.1 ubuntu.localdomain

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
127.0.2.1 ceph-node1 ceph-


################ Nodes host Name ######################
192.168.111.100 ceph-controller
192.168.111.101 ceph-node1
192.168.111.102 ceph-node2
192.168.111.103 ceph-node3
######################################################
vagrant@ceph-node1:~$ 