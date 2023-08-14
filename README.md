## Overview

Collection of simple use cases aroud macvlan on F5 XC Appstack Site. 

## Requirements

Existing F5 XC App Stack Site with at least 2 nodes with an additional Ethernet interface.
This lab setup was built using https://github.com/mwiget/f5xc-appstack-proxmox with PCI
pass through of ConnectX5 Dual 25Gbps NIC, connected back-Gto-back between 2 worker nodes:

```
$ k get nodes -o wide
NAME   STATUS   ROLES        AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                        KERNEL-VERSION                    CONTAINER-RUNTIME
m0     Ready    ves-master   24d   v1.23.14-ves   192.168.40.30   <none>        CentOS Linux 7.2009.41 (Core)   4.18.0-240.10.1.ves1.el7.x86_64   docker://19.3.12
m1     Ready    ves-master   24d   v1.23.14-ves   192.168.40.73   <none>        CentOS Linux 7.2009.41 (Core)   4.18.0-240.10.1.ves1.el7.x86_64   docker://19.3.12
m2     Ready    ves-master   24d   v1.23.14-ves   192.168.40.15   <none>        CentOS Linux 7.2009.41 (Core)   4.18.0-240.10.1.ves1.el7.x86_64   docker://19.3.12
w0     Ready    <none>       11d   v1.23.14-ves   192.168.40.83   <none>        CentOS Linux 7.2009.41 (Core)   4.18.0-240.10.1.ves1.el7.x86_64   docker://19.3.12
w1     Ready    <none>       11d   v1.23.14-ves   192.168.40.82   <none>        CentOS Linux 7.2009.41 (Core)   4.18.0-240.10.1.ves1.el7.x86_64   docker://19.3.12
```

"Prep" pods are used to activate the master network interface and to get CLI access to the node: [prep-macvlan.yaml](prep-macvlan.yaml).
Adjust according to your environment.

```
$ k apply -f prep-macvlan.yaml 
pod/prep-w0 created
pod/prep-w1 created
```

```
$ k logs prep-w0
Defaulted container "alpine" out of: alpine, wingman
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/community/x86_64/APKINDEX.tar.gz
(1/6) Installing iperf3 (3.14-r0)
(2/6) Installing hwdata-pci (0.370-r0)
(3/6) Installing pciutils-libs (3.10.0-r0)
(4/6) Installing pciutils (3.10.0-r0)
(5/6) Installing libpcap (1.10.4-r1)
(6/6) Installing tcpdump (4.99.4-r1)
Executing busybox-1.36.1-r2.trigger
OK: 10 MiB in 21 packages
00:10.0 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5]
00:10.1 Ethernet controller: Mellanox Technologies MT27800 Family [ConnectX-5]
00:12.0 Ethernet controller: Red Hat, Inc. Virtio network device
```

