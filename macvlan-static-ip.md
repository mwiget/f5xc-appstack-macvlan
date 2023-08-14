## macvlan with static ip allocation

[macvlan-static.yaml](macvlan-static.yaml) creates 2 alpine pods with individual NetworkAttachmentDefinitio's
to provide static IPv4 and IPv6 addresses. iperf3 is installed and can be used to test connectivity between
the pods.

```
+---------+                   +---------+
|   Pod   | net1         net1 |   Pod   |
| alpine1 |-------------------| alpine2 |
+---------+  25Gbps DAC link  +---------+
eth0 |                        eth0 |
    -+-----------------------------+-
             default CNI
```

|Pod      | IPv4         | IPv6        |
|---------|--------------|-------------|
| alpine1 | 10.1.0.11/24 | fd10::11/64 |
| alpine2 | 10.1.0.12/24 | fd10::12/64 |


## Deploy & Validate

```
k apply -f alpine-static.yaml

networkattachmentdefinition.k8s.cni.cncf.io/macvlan-static-alpine1 created
networkattachmentdefinition.k8s.cni.cncf.io/macvlan-static-alpine2 created
pod/alpine1 created
pod/alpine2 created
```

Check network attachment definitions:

```
$ k get network-attachment-definitions
NAME                     AGE
macvlan-static-alpine1   3m
macvlan-static-alpine2   3m
```

```
$ k describe network-attachment-definition macvlan-static-alpine1 
Name:         macvlan-static-alpine1
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  k8s.cni.cncf.io/v1
Kind:         NetworkAttachmentDefinition
Metadata:
  Creation Timestamp:  2023-08-14T10:41:19Z
  Generation:          1
  Managed Fields:
    API Version:  k8s.cni.cncf.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:config:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2023-08-14T10:41:19Z
  Resource Version:  11858701
  Self Link:         /apis/k8s.cni.cncf.io/v1/namespaces/default/network-attachment-definitions/macvlan-static-alpine1
  UID:               3bf7abb4-b9ec-4180-bb2a-bbe6cafb9772
Spec:
  Config:  { "cniVersion": "0.3.0", "type": "macvlan", "master": "ens16f0", "mode": "bridge", "ipam": { "type": "static", "addresses": [ { "address": "10.1.0.11/24" }, { "address": "fd10::11/64" } ] } }
Events:    <none>
```


Check IPv4 connectivity between pods:

```
$ k exec -ti alpine1 -- ping -c 3 10.1.0.12
Defaulted container "alpine" out of: alpine, wingman
PING 10.1.0.12 (10.1.0.12): 56 data bytes
64 bytes from 10.1.0.12: seq=0 ttl=64 time=0.201 ms
64 bytes from 10.1.0.12: seq=1 ttl=64 time=0.189 ms
64 bytes from 10.1.0.12: seq=2 ttl=64 time=0.194 ms

--- 10.1.0.12 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.189/0.194/0.201 ms
```

Check IPv6 connectivity between pods:

```
$ k exec -ti alpine1 -- ping -c 3 fd10::12
Defaulted container "alpine" out of: alpine, wingman
PING fd10::12 (fd10::12): 56 data bytes
64 bytes from fd10::12: seq=0 ttl=64 time=0.316 ms
64 bytes from fd10::12: seq=1 ttl=64 time=0.229 ms
64 bytes from fd10::12: seq=2 ttl=64 time=0.219 ms

--- fd10::12 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.219/0.254/0.316 ms
```

Check iperf3 performance between pods:

```
$ k exec -ti alpine1 -- iperf3 -c fd10::12
Defaulted container "alpine" out of: alpine, wingman
Connecting to host fd10::12, port 5201
[  5] local fd10::11 port 55436 connected to fd10::12 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  1.75 GBytes  15.0 Gbits/sec   82    791 KBytes       
[  5]   1.00-2.00   sec  2.42 GBytes  20.8 Gbits/sec   24    985 KBytes       
[  5]   2.00-3.00   sec  2.42 GBytes  20.8 Gbits/sec  173    852 KBytes       
[  5]   3.00-4.00   sec  2.27 GBytes  19.5 Gbits/sec    0   1022 KBytes       
[  5]   4.00-5.00   sec  2.16 GBytes  18.6 Gbits/sec   89    971 KBytes       
[  5]   5.00-6.00   sec  2.35 GBytes  20.2 Gbits/sec  118    720 KBytes       
[  5]   6.00-7.00   sec  2.24 GBytes  19.3 Gbits/sec    8    785 KBytes       
[  5]   7.00-8.00   sec  2.13 GBytes  18.3 Gbits/sec    0    855 KBytes       
[  5]   8.00-9.00   sec  2.22 GBytes  19.1 Gbits/sec   22    870 KBytes       
[  5]   9.00-10.00  sec  2.23 GBytes  19.1 Gbits/sec    0   1.02 MBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  22.2 GBytes  19.1 Gbits/sec  516             sender
[  5]   0.00-10.00  sec  22.2 GBytes  19.1 Gbits/sec                  receiver

iperf Done.
```

Actual results will vary based on # of cores per node, network link speed and running baremetal or virtualized. The result
captured here is between virtualized nodes on Proxmox server with a DAC loopback cable between 2 NICs directly attached to
each App Stack node Virtual Machine.

Check traffic actually went via net1 interfaces:

```
 k exec -ti alpine1 -- ifconfig
Defaulted container "alpine" out of: alpine, wingman
eth0      Link encap:Ethernet  HWaddr 06:B5:C5:56:51:26  
          inet addr:10.1.0.42  Bcast:10.1.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1436  Metric:1
          RX packets:7836 errors:0 dropped:0 overruns:0 frame:0
          TX packets:16483 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:3033638 (2.8 MiB)  TX bytes:14765561 (14.0 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

net1      Link encap:Ethernet  HWaddr BA:9C:55:1F:3E:5E  
          inet addr:10.1.0.11  Bcast:10.1.0.255  Mask:255.255.255.0
          inet6 addr: fe80::b89c:55ff:fe1f:3e5e/64 Scope:Link
          inet6 addr: fd10::11/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:609075 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1226737 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:44420270 (42.3 MiB)  TX bytes:77983008207 (72.6 GiB)
```

## Destroy pods

```
$ k delete -f alpine-static.yaml
```

L
