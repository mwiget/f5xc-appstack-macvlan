## macvlan with dynamic ip allocation

[macvlan-w0-ens16f0.yaml](macvlan-w0-ens16f0.yaml) and [macvlan-w1-ens16f0.yaml](macvlan-w1-ens16f0.yaml)
create two network attachment definitions for the same interface and network prefixes (v4 & v6), but 
different ranges. Multus IPAM allocation is a "node local matter". For pods to communicate over macvlan
between nodes, their dynamically allocated IP addresses must be unique.

[alpine-dynamic.yaml](alpine-dynamic.yaml) creates pod alpine1 and alpine2, attached to separate
macvlan network attachments via annotation.


```
  node w0:                      node w1:

+---------+                   +---------+
|   Pod   | net1         net1 |   Pod   |
| alpine1 |-------------------| alpine2 |
+---------+  25Gbps DAC link  +---------+
eth0 |                        eth0 |
    -+-----------------------------+-
             default CNI
```

## Deploy & Validate

Create network attachement definitions

```
$ k create -f macvlan-w0-ens16f0.yaml -f macvlan-w1-ens16f0.yaml 
networkattachmentdefinition.k8s.cni.cncf.io/macvlan-w0-ens16f0 created
networkattachmentdefinition.k8s.cni.cncf.io/macvlan-w1-ens16f0 created
```

```
$ k get network-attachment-definition
NAME                 AGE
macvlan-w0-ens16f0   20s
macvlan-w1-ens16f0   19s
```

Check details for one of them

```
$ k describe network-attachment-definition macvlan-w0-ens16f0
Name:         macvlan-w0-ens16f0
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  k8s.cni.cncf.io/v1
Kind:         NetworkAttachmentDefinition
Metadata:
  Creation Timestamp:  2023-08-14T13:56:45Z
  Generation:          1
  Managed Fields:
    API Version:  k8s.cni.cncf.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        .:
        f:config:
    Manager:         kubectl-create
    Operation:       Update
    Time:            2023-08-14T13:56:45Z
  Resource Version:  11928224
  Self Link:         /apis/k8s.cni.cncf.io/v1/namespaces/default/network-attachment-definitions/macvlan-w0-ens16f0
  UID:               2847f6c0-50e1-41b4-8e3a-68b436fc34e1
Spec:
  Config:  { "cniVersion": "0.3.0", "type": "macvlan", "master": "ens16f0", "mode": "bridge", "ipam": { "type": "host-local", "ranges": [ [ { "subnet": "10.1.10.0/24", "rangeStart": "10.1.10.10", "rangeEnd": "10.1.10.19" } ], [ { "subnet": "fd10::/64", "rangeStart": "fd10::10", "rangeEnd": "fd10::19" } ] ] } }
Events:    <none>
```

Deploy pods

```
$ k apply -f alpine-dynamic.yaml 
pod/alpine1 created
pod/alpine2 created
```

Check where the pods are running:

```
$ k get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP          NODE   NOMINATED NODE   READINESS GATES
alpine1   2/2     Running   0          15m   10.1.0.44   w0     <none>           <none>
alpine2   2/2     Running   0          15m   10.1.0.45   w1     <none>           <none>
```

Get ip addressses from alpine2 pod net1 (macvlan interface):

```
$ k exec -ti alpine2 -- ifconfig net1
Defaulted container "alpine" out of: alpine, wingman
net1      Link encap:Ethernet  HWaddr AE:72:93:85:D4:4D
          inet addr:10.1.10.22  Bcast:10.1.10.255  Mask:255.255.255.0
          inet6 addr: fd10::22/64 Scope:Global
          inet6 addr: fe80::ac72:93ff:fe85:d44d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:16 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:1324 (1.2 KiB)

```

Check IPv4 connectivity between pods:

```
$ k exec -ti alpine1 -- ping -c 3 10.1.10.22

Defaulted container "alpine" out of: alpine, wingman
PING 10.1.10.22 (10.1.10.22): 56 data bytes
64 bytes from 10.1.10.22: seq=0 ttl=64 time=0.753 ms
64 bytes from 10.1.10.22: seq=1 ttl=64 time=0.196 ms
64 bytes from 10.1.10.22: seq=2 ttl=64 time=0.184 ms

--- 10.1.10.22 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.184/0.377/0.753 ms
```

Check IPv6 connectivity between pods:

```
$ k exec -ti alpine1 -- ping -c 3 fd10::22
Defaulted container "alpine" out of: alpine, wingman
PING fd10::22 (fd10::22): 56 data bytes
64 bytes from fd10::22: seq=0 ttl=64 time=0.818 ms
64 bytes from fd10::22: seq=1 ttl=64 time=0.160 ms
64 bytes from fd10::22: seq=2 ttl=64 time=0.155 ms

--- fd10::22 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.155/0.377/0.818 ms
```

Check iperf3 performance between pods:
1G
```
$ k exec -ti alpine1 -- iperf3 -c fd10::22

Defaulted container "alpine" out of: alpine, wingman
Connecting to host fd10::22, port 5201
[  5] local fd10::12 port 48598 connected to fd10::22 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  2.17 GBytes  18.6 Gbits/sec  226    904 KBytes       
[  5]   1.00-2.00   sec  2.12 GBytes  18.2 Gbits/sec   57    814 KBytes       
[  5]   2.00-3.00   sec  2.10 GBytes  18.0 Gbits/sec    0    932 KBytes       
[  5]   3.00-4.00   sec  2.13 GBytes  18.3 Gbits/sec    5    757 KBytes       
[  5]   4.00-5.00   sec  2.21 GBytes  19.0 Gbits/sec    0    892 KBytes       
[  5]   5.00-6.00   sec  2.14 GBytes  18.3 Gbits/sec   60    842 KBytes       
[  5]   6.00-7.00   sec  2.42 GBytes  20.8 Gbits/sec   26    799 KBytes       
[  5]   7.00-8.00   sec  1.96 GBytes  16.8 Gbits/sec   64    841 KBytes       
[  5]   8.00-9.00   sec  2.14 GBytes  18.4 Gbits/sec    0   1005 KBytes       
[  5]   9.00-10.00  sec  2.13 GBytes  18.3 Gbits/sec  176    909 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  21.5 GBytes  18.5 Gbits/sec  614             sender
[  5]   0.00-10.00  sec  21.5 GBytes  18.5 Gbits/sec                  receiver

iperf Done.
```

Actual results will vary based on # of cores per node, network link speed and running baremetal or virtualized. The result
captured here is between virtualized nodes on Proxmox server with a DAC loopback cable between 2 NICs directly attached to
each App Stack node Virtual Machine.

Check traffic actually went via net1 interfaces:

```
$ k exec -ti alpine1 -- ifconfig
 
Defaulted container "alpine" out of: alpine, wingman
eth0      Link encap:Ethernet  HWaddr 72:1C:FA:D7:E5:F0  
          inet addr:10.1.0.46  Bcast:10.1.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1436  Metric:1
          RX packets:2137 errors:0 dropped:0 overruns:0 frame:0
          TX packets:964 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:2528168 (2.4 MiB)  TX bytes:259299 (253.2 KiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

net1      Link encap:Ethernet  HWaddr EA:FD:CB:C2:35:40  
          inet addr:10.1.10.14  Bcast:10.1.10.255  Mask:255.255.255.0
          inet6 addr: fe80::e8fd:cbff:fec2:3540/64 Scope:Link
          inet6 addr: fd10::12/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:173938 errors:0 dropped:0 overruns:0 frame:0
          TX packets:369492 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:14959692 (14.2 MiB)  TX bytes:23126289711 (21.5 GiB)

```

## Destroy pods

```
$ k delete -f alpine-dynamic.yaml
```
