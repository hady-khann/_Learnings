| Release series | Latest version |
| -------------- | -------------: |
| **Tentacle**   |         20.2.2 |
| **Squid**      |         19.2.5 |
| **Reef**       |         18.2.8 |
| **Quincy**     |         17.2.9 |
| **Pacific**    |        16.2.15 |
| **Octopus**    |        15.2.17 |
| **Nautilus**   |        14.2.22 |
| **Mimic**      |        13.2.10 |
| **Luminous**   |        12.2.13 |
| **Kraken**     |         11.2.1 |



#########################################
# install
#########################################
sudo apt update
sudo apt install -y cephadm ceph-common
cephadm version

cephadm version
ceph --version

#########################################
Bootstrap the cluster
#########################################
sudo cephadm bootstrap --mon-ip <your-first-monitor-ip>