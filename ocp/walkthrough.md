# 离线安装

## 网络设置

## 准备安装源

## 安装Helper节点

## Advanced Networking
```
(undercloud) [stack@undercloud ~]$ openstack server list
+--------------------------------------+-------------------------+--------+---------------------+----------------+---------+
| ID                                   | Name                    | Status | Networks            | Image          | Flavor  |
+--------------------------------------+-------------------------+--------+---------------------+----------------+---------+
| 7083d62e-9eb4-436b-9ceb-a3e8eec9947d | overcloud-controller-1  | ACTIVE | ctlplane=192.0.2.18 | overcloud-full | control |
| 13adde5a-5439-4248-8f0c-af709c1c5683 | overcloud-controller-0  | ACTIVE | ctlplane=192.0.2.17 | overcloud-full | control |
| 2758123f-6b04-4887-b47c-6fe8521cccc7 | overcloud-controller-2  | ACTIVE | ctlplane=192.0.2.8  | overcloud-full | control |
| 3c8172e1-d17c-49fa-84e0-0573c7b11a36 | overcloud-novacompute-1 | ACTIVE | ctlplane=192.0.2.21 | overcloud-full | compute |
| efab2bf9-5777-467d-8eab-094151c862d1 | overcloud-novacompute-0 | ACTIVE | ctlplane=192.0.2.15 | overcloud-full | compute |
+--------------------------------------+-------------------------+--------+---------------------+----------------+---------+


```

```
# 3.1. Student tasks
## Create VMs named vm1vx/vm1gre and vm2vx/vm1gre (using vmname.img copying from cirros-nonetwork.img)


## Copy image to /var/lib/libvirt/images on rhel01

[root@rhel01 ]# cd /var/lib/libvirt/images
[root@rhel01 images]# curl -O http://www.opentlc.com/download/osp_advanced_networking/cirros-nonetwork.img
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 28.5M  100 28.5M    0     0  5084k      0  0:00:05  0:00:05 --:--:-- 6229k
[root@rhel01 images]# cp cirros-nonetwork.img /var/lib/libvirt/images/vm1vx.img
[root@rhel01 images]# cp cirros-nonetwork.img /var/lib/libvirt/images/vm1gre.img
[root@rhel01 images]# chown qemu:qemu /var/lib/libvirt/images/*.img
[root@rhel01 images]# ll
total 116864
-rw-r--r--. 1 qemu qemu 29884416 Jul 20 04:35 cirros-nonetwork.img
-rw-r--r--. 1 qemu qemu 30015488 Jul 20 02:02 vm1.img
-rw-r--r--. 1 qemu qemu 29884416 Jul 20 04:36 vm1gre.img
-rw-r--r--. 1 qemu qemu 29884416 Jul 20 04:36 vm1vx.img

## Copy image to /var/lib/libvirt/images on rhel02
[root@rhel02 ~]# cd /var/lib/libvirt/images
[root@rhel02 images]# curl -O http://www.opentlc.com/download/osp_advanced_networking/cirros-nonetwork.img
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 28.5M  100 28.5M    0     0  5085k      0  0:00:05  0:00:05 --:--:-- 6528k
[root@rhel02 images]# cp cirros-nonetwork.img /var/lib/libvirt/images/vm2vx.img
[root@rhel02 images]# cp cirros-nonetwork.img /var/lib/libvirt/images/vm2gre.img
[root@rhel02 images]# chown qemu:qemu /var/lib/libvirt/images/*.img
[root@rhel02 images]# ll
total 116864
-rw-r--r--. 1 qemu qemu 29884416 Jul 20 04:38 cirros-nonetwork.img
-rw-r--r--. 1 qemu qemu 30015488 Jul 20 02:03 vm2.img
-rw-r--r--. 1 qemu qemu 29884416 Jul 20 04:38 vm2gre.img
-rw-r--r--. 1 qemu qemu 29884416 Jul 20 04:38 vm2vx.img

## Ensure you are using bridges br-vx and br-gre
## 
[root@rhel01 images]# ovs-vsctl add-br br-vx                       
[root@rhel01 images]# ovs-vsctl add-port br-vx nic1 -- set interface nic1 type=internal
[root@rhel01 images]# ovs-vsctl add-port br-vx vx0 -- set interface vx0 type=vxlan options:remote_ip=192.0.2.246
[root@rhel01 images]# ovs-vsctl add-br br-gre
[root@rhel01 images]# ovs-vsctl add-port br-gre nic2 -- set interface nic2 type=internal
[root@rhel01 images]# ovs-vsctl add-port br-gre gre0 -- set interface gre0 type=gre options:remote_ip=192.0.2.246

[root@rhel02 images]# ovs-vsctl add-br br-vx 
[root@rhel02 images]# ovs-vsctl add-port br-vx nic1 -- set interface nic1 type=internal
[root@rhel02 images]# ovs-vsctl add-port br-vx vx0 -- set interface vx0 type=vxlan options:remote_ip=192.0.2.245
[root@rhel02 images]# ovs-vsctl add-br br-gre
[root@rhel02 images]# ovs-vsctl add-port br-gre nic2 -- set interface nic2 type=internal
[root@rhel02 images]# ovs-vsctl add-port br-gre gre0 -- set interface gre0 type=gre options:remote_ip=192.0.2.245

# make sure link is up on rhel01 and rhel02
[root@rhel01 images]# ip link set br-vx up 
[root@rhel01 images]# ip link set br-gre up 
[root@rhel01 images]# ip link set nic1 up 
[root@rhel01 images]# ip link set nic2 up 

[root@rhel02 images]# ip link set br-vx up 
[root@rhel02 images]# ip link set br-gre up
[root@rhel02 images]# ip link set nic1 up
[root@rhel02 images]# ip link set nic2 up

# Create VM vm1vx on rhel01
[root@rhel01 images]# virt-install --ram 128 --vcpus 1 --os-type linux --disk path=/var/lib/libvirt/images/vm1vx.img,device=disk,bus=virtio,format=qcow2 --import --noautoconsole --vnc --network bridge:br-vx,model=virtio,virtualport_type=openvswitch --name vm1vx
WARNING  OS name 'linux' is deprecated, using 'generic' instead. This alias will be removed in the future.

Starting install...
Domain creation completed.

# Create VM vm1gre on rhel01
[root@rhel01 images]# virt-install --ram 128 --vcpus 1 --os-type linux --disk path=/var/lib/libvirt/images/vm1gre.img,device=disk,bus=virtio,format=qcow2 --import --noautoconsole --vnc --network bridge:br-gre,model=virtio,virtualport_type=openvswitch --name vm1gre
WARNING  OS name 'linux' is deprecated, using 'generic' instead. This alias will be removed in the future.

Starting install...
Domain creation completed.

# Create VM vm2vx on rhel02
[root@rhel02 images]# virt-install --ram 128 --vcpus 1 --os-type linux --disk path=/var/lib/libvirt/images/vm2vx.img,device=disk,bus=virtio,format=qcow2 --import --noautoconsole --vnc --network bridge:br-vx,model=virtio,virtualport_type=openvswitch --name vm2vx
WARNING  OS name 'linux' is deprecated, using 'generic' instead. This alias will be removed in the future.

Starting install...
Domain creation completed.

# Create VM vm2gre on rhel02
[root@rhel02 images]# virt-install --ram 128 --vcpus 1 --os-type linux --disk path=/var/lib/libvirt/images/vm2gre.img,device=disk,bus=virtio,format=qcow2 --import --noautoconsole --vnc --network bridge:br-gre,model=virtio,virtualport_type=openvswitch --name vm2gre
WARNING  OS name 'linux' is deprecated, using 'generic' instead. This alias will be removed in the future.

Starting install...
Domain creation completed.

## Use range 192.168.101.0/24 and 192.168.102.0/24
# Connect to vm1vx and configure ip 192.168.101.30 on rhel01
[root@rhel01 images]# virsh console vm1vx 
Connected to domain vm1vx
Escape character is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
cirros login: cirros
Password: 
$ sudo ifconfig eth0 192.168.101.30 netmask 255.255.255.0 up
# Escape from vm1vx by typing ^]

# Connect to vm1gre and configure ip 192.168.102.30 on rhel01
[root@rhel01 images]# virsh console vm1gre
Connected to domain vm1gre
Escape character is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
cirros login: cirros
Password: 
$ sudo ifconfig eth0 192.168.102.30 netmask 255.255.255.0 up
# Escape from vm1vx by typing ^]

# Connect to vm2vx and configure ip 192.168.101.31 on rhel02
[root@rhel02 images]# virsh console vm2vx
Connected to domain vm2vx
Escape character is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
cirros login: cirros
Password: 
$ sudo ifconfig eth0 192.168.101.31 netmask 255.255.255.0 up
# Escape from vm1vx by typing ^]

# Connect to vm2gre and configure ip 192.168.102.31 on rhel02
[root@rhel02 images]# virsh console vm2gre
Connected to domain vm2gre
Escape character is ^]

cirros login: cirros
Password: 
$ sudo ifconfig eth0 192.168.102.31 netmask 255.255.255.0 up

# Try ping communication between VMs
# vm1vx(192.168.101.30) to vm2vx(192.168.101.31) on rhel01 
[root@rhel01 images]# virsh console vm1vx
Connected to domain vm1vx
Escape character is ^]

$ ping -c1 192.168.101.31 
PING 192.168.101.31 (192.168.101.31): 56 data bytes
64 bytes from 192.168.101.31: seq=0 ttl=64 time=2.250 ms

--- 192.168.101.31 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 2.250/2.250/2.250 ms

# vm1gre(192.168.102.30) to vm2gre(192.168.102.31) on rhel01 
[root@rhel01 images]# virsh console vm1gre
Connected to domain vm1gre
Escape character is ^]

$ ping -c1 192.168.102.31
PING 192.168.102.31 (192.168.102.31): 56 data bytes
64 bytes from 192.168.102.31: seq=0 ttl=64 time=1.726 ms

--- 192.168.102.31 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.726/1.726/1.726 ms

# Create VMs vm10vx on rhel01
[root@rhel01 images]# cd /var/lib/libvirt/images
[root@rhel01 images]# cp cirros-nonetwork.img vm10vx.img
[root@rhel01 images]# virt-install --ram 128 --vcpus 1 --os-type linux --disk path=/var/lib/libvirt/images/vm10vx.img,device=disk,bus=virtio,format=qcow2 --import --noautoconsole --vnc --network bridge:br-vx,model=virtio,virtualport_type=openvswitch --name vm10vx
WARNING  OS name 'linux' is deprecated, using 'generic' instead. This alias will be removed in the future.

Starting install...
Domain creation completed.

# Set vm10vx ip address 192.168.101.40/24
[root@rhel01 images]# virsh console vm10vx
Connected to domain vm10vx
Escape character is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
cirros login: cirros
Password: 
$ sudo ifconfig eth0 192.168.101.40 netmask 255.255.255.0 up
$ 
# Escape by type ^]


# Create VMs vm20vx on rhel02
cd /var/lib/libvirt/images
cp cirros-nonetwork.img vm20vx.img
virt-install --ram 128 --vcpus 1 --os-type linux --disk path=/var/lib/libvirt/images/vm20vx.img,device=disk,bus=virtio,format=qcow2 --import --noautoconsole --vnc --network bridge:br-vx,model=virtio,virtualport_type=openvswitch --name vm20vx

[root@rhel02 images]# virsh console vm20vx
Connected to domain vm20vx
Escape character is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
cirros login: cirros
Password: 
$ sudo ifconfig eth0 192.168.101.41 netmask 255.255.255.0 up

# Check communication between 4 VMs
# from vm20vx(192.168.101.41) ping vm10vx(192.168.101.40)
[root@rhel02 images]# virsh console vm20vx
Connected to domain vm20vx
Escape character is ^]

$ ping -c1 192.168.101.40
PING 192.168.101.40 (192.168.101.40): 56 data bytes
64 bytes from 192.168.101.40: seq=0 ttl=64 time=1.447 ms

--- 192.168.101.40 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.447/1.447/1.447 ms

# from vm20vx(192.168.101.41) ping vm2vx(192.168.101.31)
[root@rhel02 images]# virsh console vm20vx
Connected to domain vm20vx
Escape character is ^]

$ ping -c1 192.168.101.31
PING 192.168.101.31 (192.168.101.31): 56 data bytes
64 bytes from 192.168.101.31: seq=0 ttl=64 time=1.446 ms

--- 192.168.101.31 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.446/1.446/1.446 ms

# from vm20vx(192.168.101.41) ping vm1vx(192.168.101.30)
[root@rhel02 images]# virsh console vm20vx
Connected to domain vm20vx
Escape character is ^]

$ ping -c1 192.168.101.30
PING 192.168.101.30 (192.168.101.30): 56 data bytes
64 bytes from 192.168.101.30: seq=0 ttl=64 time=1.453 ms

--- 192.168.101.30 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.453/1.453/1.453 ms

# Use ovs-vsctl set port vnetX tag=10 for the virtual ports of new VMs
[root@rhel01 images]# virsh domiflist vm10vx 
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet2      bridge     br-vx      virtio      52:54:00:44:a1:14
[root@rhel01 images]# ovs-vsctl set port vnet2 tag=10 

[root@rhel02 images]# virsh domiflist vm20vx
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet2      bridge     br-vx      virtio      52:54:00:f7:4d:de
[root@rhel02 images]# ovs-vsctl set port vnet2 tag=10 

# Check only communication between different servers are working.

# from vm20vx ping vm10vx should work because these vms are in same vlan tag=10
[root@rhel02 images]# virsh console vm20vx
Connected to domain vm20vx
Escape character is ^]

$ ping -c1 192.168.101.40 
PING 192.168.101.40 (192.168.101.40): 56 data bytes
64 bytes from 192.168.101.40: seq=0 ttl=64 time=4.375 ms

--- 192.168.101.40 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 4.375/4.375/4.375 ms

# from vm20vx(192.168.101.41,vlan tag 10) ping vm1vx(192.168.101.30, no vlan tag) and from vm20vx(192.168.101.41,vlan tag 10) ping vm2vx(192.168.101.31, no vlan tag) should not work because these vm are not in same vlan
[root@rhel02 images]# virsh console vm20vx
Connected to domain vm20vx
Escape character is ^]

$ ping -c1 -W1 192.168.101.30 
PING 192.168.101.30 (192.168.101.30): 56 data bytes

--- 192.168.101.30 ping statistics ---
1 packets transmitted, 0 packets received, 100% packet loss
$ ping -c1 -W1 192.168.101.31 
PING 192.168.101.31 (192.168.101.31): 56 data bytes

--- 192.168.101.31 ping statistics ---
1 packets transmitted, 0 packets received, 100% packet loss

# Remove the vms and undefine them.
[root@rhel01 images]# virsh destroy vm1vx  
Domain vm1vx destroyed

[root@rhel01 images]# virsh destroy vm1gre
Domain vm1gre destroyed

[root@rhel01 images]# virsh destroy vm10vx
Domain vm10vx destroyed

[root@rhel01 images]# virsh undefine vm1vx
Domain vm1vx has been undefined

[root@rhel01 images]# virsh undefine vm1gre
Domain vm1gre has been undefined

[root@rhel01 images]# virsh undefine vm10vx
Domain vm10vx has been undefined

[root@rhel02 images]# for i in vm2vx vm2gre vm20vx ; do virsh destroy $i ; virsh undefine $i ; done 
Domain vm2vx destroyed

Domain vm2vx has been undefined

Domain vm2gre destroyed

Domain vm2gre has been undefined

Domain vm20vx destroyed

Domain vm20vx has been undefined

```

```
Using Neutron
1. Check OpenStack Networking Services
2. Review enabled Network Types
3. Create a Network Environment with Neutron CLI Commands
3.1. Create a Router
3.2. Obtain Information about Neutron Ports
3.3. Operate with Floating IP
3.4. Work with ports
3.5. Disable port security: ports without security group
```

```
Classic Deployment with Open vSwitch Lab
1. Prepare the Environment
2. Review Wiring Inside Compute Node
3. North-South Traffic
4. Packet Tracing and Switching
4.1. Switching
4.2. Logical Tracing
5. Accessing OVN database content
5.1. Northbound database
5.2. Southbound database

overcloud-novacompute-0.localdomain
instance-00000006

+--------------------------------------+-------------------------+--------+---------------------+----------------+---------+
| ID                                   | Name                    | Status | Networks            | Image          | Flavor  |
+--------------------------------------+-------------------------+--------+---------------------+----------------+---------+
| 7083d62e-9eb4-436b-9ceb-a3e8eec9947d | overcloud-controller-1  | ACTIVE | ctlplane=192.0.2.18 | overcloud-full | control |
| 13adde5a-5439-4248-8f0c-af709c1c5683 | overcloud-controller-0  | ACTIVE | ctlplane=192.0.2.17 | overcloud-full | control |
| 2758123f-6b04-4887-b47c-6fe8521cccc7 | overcloud-controller-2  | ACTIVE | ctlplane=192.0.2.8  | overcloud-full | control |
| 3c8172e1-d17c-49fa-84e0-0573c7b11a36 | overcloud-novacompute-1 | ACTIVE | ctlplane=192.0.2.21 | overcloud-full | compute |
| efab2bf9-5777-467d-8eab-094151c862d1 | overcloud-novacompute-0 | ACTIVE | ctlplane=192.0.2.15 | overcloud-full | compute |
+--------------------------------------+-------------------------+--------+---------------------+----------------+---------+

ssh heat-admin@192.0.2.15

[root@overcloud-novacompute-0 ~]# virsh dumpxml instance-00000006 | grep interface -A10

    <interface type='bridge'>
      <mac address='fa:fa:fa:d0:d0:d0'/>
      <source bridge='br-int'/>
      <virtualport type='openvswitch'>
        <parameters interfaceid='476212e4-4c1b-46c2-a4af-c1ddefba9215'/>
      </virtualport>
      <target dev='tap476212e4-4c'/>
      <model type='virtio'/>
      <driver name='vhost' rx_queue_size='512'/>
      <mtu size='1442'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>

[heat-admin@overcloud-controller-2 ~]$ sudo podman exec -it ovn-dbs-bundle-podman-0 ovn-sbctl find Port_Binding logical_port=476212e4-4c1b-46c2-a4af-c1ddefba9215 
_uuid               : b0fd65c9-f279-4182-9eb5-60749e6f56fd
chassis             : 6e7255ea-e9b3-474e-a9be-ee6370a7a43f
datapath            : 368ff0e6-8c02-40d6-98f6-242342e39adc
encap               : []
external_ids        : {name="vm1port", "neutron:cidrs"="192.168.100.101/24", "neutron:device_id"="ddbbb2e5-a2ba-4841-bb5a-40b256538dae", "neutron:device_owner"="compute:nova", "neutron:network_name"="neutron-3a3ab3a1-5aab-4026-89b1-0007ac0558da", "neutron:port_name"="vm1port", "neutron:project_id"="10e6f16ae2aa4c7b899e737d87d65129", "neutron:revision_number"="8", "neutron:security_group_ids"=""}
gateway_chassis     : []
ha_chassis_group    : []
logical_port        : "476212e4-4c1b-46c2-a4af-c1ddefba9215"
mac                 : ["fa:fa:fa:d0:d0:d0 192.168.100.101", unknown]
nat_addresses       : []
options             : {requested-chassis="overcloud-novacompute-0.localdomain"}
parent_port         : []
tag                 : []
tunnel_key          : 4
type                : ""
virtual_parent      : []


ovn-sbctl get Chassis $(ovn-sbctl find Port_Binding logical_port=cr-lrp-b4ecd092-0869-47d3-927b-a7ad014c62f0 | grep ^chassis | awk '{print $3}') hostname


```



```
openstack security group create sg-web
openstack security group rule create --protocol tcp --dst-port 22 sg-web
openstack security group rule create --protocol tcp --dst-port 80 sg-web

openstack server create --image cirros --flavor m1.tiny --security-group sg-web --nic net-id=private1 web01
openstack server create --image cirros --flavor m1.tiny --security-group sg-web --nic net-id=private1 web02

openstack server list

ROUTERID=$(openstack router show router1 -c id -f value)

```

```
[heat-admin@overcloud-controller-2 ~]$ ip a s | grep vlan
9: vlan10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 10.0.0.165/24 brd 10.0.0.255 scope global vlan10
10: vlan20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.17.0.200/24 brd 172.17.0.255 scope global vlan20
11: vlan40: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.19.0.92/24 brd 172.19.0.255 scope global vlan40
12: vlan30: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.18.0.122/24 brd 172.18.0.255 scope global vlan30
14: vlan50: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.16.0.33/24 brd 172.16.0.255 scope global vlan50

[heat-admin@overcloud-controller-1 ~]$ sudo ip a s | grep vlan
6: vlan20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.17.0.117/24 brd 172.17.0.255 scope global vlan20
7: vlan40: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.19.0.60/24 brd 172.19.0.255 scope global vlan40
    inet 172.19.0.123/32 brd 172.19.0.255 scope global vlan40
8: vlan10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 10.0.0.249/24 brd 10.0.0.255 scope global vlan10
    inet 10.0.0.91/32 brd 10.0.0.255 scope global vlan10
9: vlan30: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.18.0.189/24 brd 172.18.0.255 scope global vlan30
10: vlan50: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.16.0.130/24 brd 172.16.0.255 scope global vlan50

[heat-admin@overcloud-controller-0 ~]$ ip a s | grep vlan
6: vlan40: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.19.0.210/24 brd 172.19.0.255 scope global vlan40
7: vlan20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.17.0.221/24 brd 172.17.0.255 scope global vlan20
    inet 172.17.0.25/32 brd 172.17.0.255 scope global vlan20
    inet 172.17.0.164/32 brd 172.17.0.255 scope global vlan20
8: vlan10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 10.0.0.25/24 brd 10.0.0.255 scope global vlan10
10: vlan30: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.18.0.24/24 brd 172.18.0.255 scope global vlan30
    inet 172.18.0.142/32 brd 172.18.0.255 scope global vlan30
11: vlan50: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.16.0.230/24 brd 172.16.0.255 scope global vlan50



sed  '/crypto_hash: sha256/a     ' input
```

```
openstack network create private --dns-domain example.com.
openstack subnet create sub_private --network private --subnet-range 192.168.100.0/24 --dns-nameserver 8.8.8.8

curl -L http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img -O
openstack image create --disk-format qcow2 --file cirros-0.4.0-x86_64-disk.img cirros

openstack flavor create --ram 128 --disk 1 --vcpus 1 m1.tiny

sg_id=$(openstack security group list | grep $(openstack project show admin -f value -c id) | awk '{ print $2 }')
openstack security group rule create --proto icmp $sg_id
openstack security group rule create --dst-port 22 --protocol tcp $sg_id

openstack server create --image cirros --flavor m1.tiny --nic net-id=private,v4-fixed-ip=192.168.100.100 testmetadata

openstack network create public  --share --external --provider-physical-network datacentre --provider-network-type vlan --provider-segment 10

openstack subnet create public --no-dhcp --network public --subnet-range 10.0.0.0/24   --allocation-pool start=10.0.0.71,end=10.0.0.200 --gateway 10.0.0.1 --dns-nameserver 8.8.8.8

openstack router create router_private

openstack router set router_private --external-gateway public

openstack router add subnet router_private sub_private

# openstack port list --device-owner compute:nova

openstack floating ip create --floating-ip-address 10.0.0.70 public

openstack server add floating ip testmetadata 10.0.0.70

openstack port create  --network private --fixed-ip subnet=sub_private,ip-address=192.168.100.101 --mac-address fa:fa:fa:d0:d0:d0 vm1port

openstack server create --image cirros --flavor m1.tiny --nic port-id=vm1port vm1 --wait

openstack floating ip create public --floating-ip-address 10.0.0.71

openstack server add floating ip vm1 10.0.0.71

openstack server ssh testmetadata --login cirros

openstack flavor create --ram 2048 --disk 20 --vcpus 1 m1.small

openstack server add security group e813faec-2236-41b9-a22d-4cd7ed842212 $sg_id

```

```
  Attributes: additional_parameters=--open-files-limit=16384 cluster_host_map=overcloud-controller-0:overcloud-controller-0.internalapi.localdomain;overcloud-controller-1:overcloud-controller-1.internalapi.localdomain;overcloud-controller-2:overcloud-controller-2.internalapi.localdomain enable_creation=true log=/var/log/mysql/mysqld.log wsrep_cluster_address=gcomm://overcloud-controller-0.internalapi.localdomain,overcloud-controller-1.internalapi.localdomain,overcloud-controller-2.internalapi.localdomain

pcs resource update galera wsrep_cluster_address=gcomm://overcloud-controller-1.internalapi.localdomain,overcloud-controller-2.internalapi.localdomain

crm_attribute -N overcloud-controller-2 -l reboot --name galera-last-committed -Q
crm_attribute -N overcloud-controller-2 -l reboot --name galera-bootstrap -v true
crm_attribute -N overcloud-controller-2 -l reboot --name master-galera -v 100
crm_resource --force-promote -r galera -V


openstack network create storage  --share --provider-physical-network datacentre --provider-network-type vlan --provider-segment 30

openstack subnet create storage --network storage --subnet-range 172.16.100.0/24   --allocation-pool start=172.16.100.100,end=172.16.100.150 --gateway 172.16.100.1 --dns-nameserver 8.8.8.8

openstack port create  --network private --fixed-ip subnet=sub_private,ip-address=192.168.100.102 --mac-address fa:fa:fa:d0:d0:d1 vm2port

openstack server create --image cirros --flavor m1.tiny --key-name admin-key --nic port-id=vm2port rh1

openstack server add port rh1 $p1_id

openstack server ssh testmetadata --login cirros

sudo ip link add link eth1 name eth1.1070 type vlan id 1070

openstack security group create sg-web
openstack security group rule create --protocol tcp --dst-port 22 sg-web
openstack security group rule create --protocol tcp --dst-port 80 sg-web
openstack security group rule create --protocol icmp sg-web

cat > webserver.sh << 'EOF'
#!/bin/sh

MYIP=$(/sbin/ifconfig eth0|grep 'inet addr'|awk -F: '{print $2}'| awk '{print $1}');
OUTPUT_STR="Welcome to $MYIP\r"
OUTPUT_LEN=${#OUTPUT_STR}

while true; do
    echo -e "HTTP/1.0 200 OK\r\nContent-Length: ${OUTPUT_LEN}\r\n\r\n${OUTPUT_STR}" | sudo nc -l -p 80
done
EOF

openstack server create --image cirros --flavor m1.tiny --security-group sg-web --nic net-id=private --user-data webserver.sh web01  --wait
openstack server create --image cirros --flavor m1.tiny --security-group sg-web --nic net-id=private --user-data webserver.sh web02  --wait

openstack server list --name web

openstack loadbalancer create --name lbweb --vip-subnet-id sub_private

openstack loadbalancer list

openstack loadbalancer listener create --name listenerweb --protocol HTTP --protocol-port 80 lbweb

openstack loadbalancer pool create --name poolweb --protocol HTTP  --listener listenerweb --lb-algorithm ROUND_ROBIN

IPWEB01=$(openstack server show web01 -c addresses -f value | cut -d"=" -f2)
IPWEB02=$(openstack server show web02 -c addresses -f value | cut -d"=" -f2)
openstack loadbalancer member create --name web01 --address $IPWEB01 --protocol-port 80 poolweb
openstack loadbalancer member create --name web02 --address $IPWEB02 --protocol-port 80 poolweb

VIP=$(openstack loadbalancer show lbweb -c vip_address -f value)
PORTID=$(openstack port list --fixed-ip ip-address=$VIP -c ID -f value)
openstack floating ip create --port $PORTID public

openstack loadbalancer amphora list

openstack server list --project service

openstack catalog show octavia


# Qos
openstack network qos policy create bw-limiter
openstack network qos rule create  --max-kbps 3000 --max-burst-kbits 300 bw-limiter --type bandwidth-limit --ingress

openstack server list --name vm1
openstack port list --server vm1
openstack port list --server vm1 -f value -c ID
openstack port set --qos-policy bw-limiter $(openstack port list --server vm1 -f value -c ID)
openstack port show $(openstack port list --server vm1 -f value -c ID)

openstack server create --image cirros --flavor m1.tiny  --nic net-id=private  testdns
openstack port show $(openstack port list -c ID -f value --server testdns)

openstack network create provider --provider-physical-network datacentre --provider-network-type vlan --provider-segment 100
openstack subnet create provider --dhcp --network provider --subnet-range 192.168.50.0/24   --allocation-pool start=192.168.50.100,end=192.168.50.200 --gateway 192.168.50.1 --dns-nameserver 8.8.8.8

nmcli con add type vlan ifname vlan100 dev eth1 id 100 ipv4.method 'manual' ipv4.address '192.168.50.50/24' 

openstack server create --image cirros --flavor m1.tiny --nic net-id=provider rh2

(overcloud) [stack@undercloud ~]$ openstack network create provider --provider-physical-network datacentre --provider-network-type vlan --provider-segment 100
(overcloud) [stack@undercloud ~]$ openstack subnet create provider --dhcp --network provider --subnet-range 192.168.50.0/24   --allocation-pool start=192.168.50.100,end=192.168.50.200 --gateway 192.168.50.1 --dns-nameserver 8.8.8.8
[root@workstation-dc36 ~]# nmcli con add type vlan ifname vlan100 dev eth1 id 100 ipv4.method 'manual' ipv4.address '192.168.50.50/24' 
Connection 'vlan-vlan100' (d7e3fb84-585d-4af7-9034-7f101f7d6a53) successfully added.
[root@workstation-dc36 ~]# nmcli c s 
NAME          UUID                                  TYPE      DEVICE  
System eth0   5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03  ethernet  eth0    
System eth1   9c92fad9-6ecb-3e6c-eb4d-8a47c6f50c04  ethernet  eth1    
vlan-vlan100  d7e3fb84-585d-4af7-9034-7f101f7d6a53  vlan      vlan100 
[root@workstation-dc36 ~]# ip a s 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 2c:c2:60:29:83:44 brd ff:ff:ff:ff:ff:ff
    inet 192.0.2.252/24 brd 192.0.2.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::2ec2:60ff:fe29:8344/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 2c:c2:60:1b:2c:d2 brd ff:ff:ff:ff:ff:ff
    inet 1.0.0.19/8 brd 1.255.255.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::2ec2:60ff:fe1b:2cd2/64 scope link 
       valid_lft forever preferred_lft forever
4: vlan100@eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 2c:c2:60:1b:2c:d2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.50/24 brd 192.168.50.255 scope global noprefixroute vlan100
       valid_lft forever preferred_lft forever
    inet6 fe80::9a2d:16ba:aac7:4bf6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

(overcloud) [stack@undercloud ~]$ openstack server create --image cirros --flavor m1.tiny --nic net-id=provider rh2
(overcloud) [stack@undercloud ~]$ openstack server list --name rh2
+--------------------------------------+------+--------+-------------------------+--------+--------+
| ID                                   | Name | Status | Networks                | Image  | Flavor |
+--------------------------------------+------+--------+-------------------------+--------+--------+
| d53e56b6-8ff4-453d-ae2f-ae9911f084cf | rh2  | ACTIVE | provider=192.168.50.141 | cirros |        |
+--------------------------------------+------+--------+-------------------------+--------+--------+
[root@workstation-dc36 ~]# ssh cirros@192.168.50.141
The authenticity of host '192.168.50.141 (192.168.50.141)' can't be established.
ECDSA key fingerprint is SHA256:I43S5Q02AliFZCyl+hmljeBrq6sRqGhpi+qgvZmY7jk.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.50.141' (ECDSA) to the list of known hosts.
cirros@192.168.50.141's password: 
$ 

```


```
(undercloud) [stack@undercloud ~]$ openstack server list
+--------------------------------------+----------------------------+--------+-------------------------+----------------+---------------+
| ID                                   | Name                       | Status | Networks                | Image          | Flavor        |
+--------------------------------------+----------------------------+--------+-------------------------+----------------+---------------+
| 26bb56b9-357e-4b39-bbb7-e9cf480aee9c | overcloud-controller-0     | ACTIVE | ctlplane=192.168.10.56  | overcloud-full | control0      |
| deb46580-cfa8-4ea2-bd11-aa9c7385fe42 | overcloud-controller-1     | ACTIVE | ctlplane=192.168.10.82  | overcloud-full | control0      |
| 61793a0f-cd84-4066-a981-c821c2fb2912 | overcloud-controller-2     | ACTIVE | ctlplane=192.168.10.29  | overcloud-full | control0      |
| a7864d10-28e5-47f7-af19-757a598f2f28 | overcloud-novacompute0-0   | ACTIVE | ctlplane=192.168.10.81  | overcloud-full | compute_leaf0 |
| 1a804c1a-5801-4c9d-bc59-6512b2dd2cd6 | overcloud-novacompute2-0   | ACTIVE | ctlplane=192.168.12.43  | overcloud-full | compute_leaf2 |
| 29aef523-d5fe-4c86-abf5-fcb5446fda8f | overcloud-novacomputeaz1-0 | ACTIVE | ctlplane=192.168.100.88 | overcloud-full | compute_az1   |
| 77a6f3fb-61a5-4ea9-ab6b-dc17d3f7b30a | overcloud-novacompute1-0   | ACTIVE | ctlplane=192.168.11.52  | overcloud-full | compute_leaf1 |
+--------------------------------------+----------------------------+--------+-------------------------+----------------+---------------+

source overcloudrc
openstack network create public --share --external --provider-physical-network datacentre   --provider-network-type vlan --provider-segment 10
openstack subnet create public --no-dhcp --network public --subnet-range 10.0.0.0/24   --allocation-pool start=10.0.0.100,end=10.0.0.200 --gateway 10.0.0.1 --dns-nameserver 8.8.8.8

openstack network create private
openstack subnet create --network private --dns-nameserver 8.8.4.4 --gateway 172.16.1.1   --subnet-range 172.16.1.0/24 private

openstack router create router1
openstack router add subnet router1 private
openstack router set router1 --external-gateway public


curl -L -O https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
openstack image create cirros --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public

openstack flavor create m1.nano --ram 256

openstack keypair create --public-key ~/.ssh/id_rsa.pub admin-key

openstack server create private_leaf0 --availability-zone nova:overcloud-novacompute0-0.localdomain --flavor m1.nano --image cirros --network private  --key-name admin-key --wait
openstack server create private_leaf1 --availability-zone nova:overcloud-novacompute1-0.localdomain --flavor m1.nano --image cirros --network private  --key-name admin-key --wait
openstack server create private_leaf2 --availability-zone nova:overcloud-novacompute2-0.localdomain --flavor m1.nano --image cirros --network private  --key-name admin-key --wait

FP1=$(openstack floating ip create public -f value -c floating_ip_address)
FP2=$(openstack floating ip create public -f value -c floating_ip_address)
FP3=$(openstack floating ip create public -f value -c floating_ip_address)

openstack server add floating ip private_leaf0 $FP1
openstack server add floating ip private_leaf1 $FP2
openstack server add floating ip private_leaf2 $FP3

IN1=$(openstack server list -c Networks --name private_leaf0 -f value|cut -d"," -f1|cut -d"=" -f2)
IN2=$(openstack server list -c Networks --name private_leaf1 -f value|cut -d"," -f1|cut -d"=" -f2)
IN3=$(openstack server list -c Networks --name private_leaf2 -f value|cut -d"," -f1|cut -d"=" -f2)

ssh cirros@$FP1 ping -c1 $IN2
ssh cirros@$FP2 ping -c1 $IN3
ssh cirros@$FP3 ping -c1 $IN1

```


```
openstack network create provider --provider-physical-network datacentre --provider-network-type vlan --provider-segment 100
openstack subnet create provider --network provider --dhcp --subnet-range 192.168.60.0/24   --allocation-pool start=192.168.60.100,end=192.168.60.150 --gateway 192.168.60.1 --dns-nameserver 8.8.8.8

curl -L http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img -O
openstack image create --disk-format qcow2 --file cirros-0.4.0-x86_64-disk.img cirros

openstack flavor create --ram 128 --disk 1 --vcpus 1 m1.tiny

sg_id=$(openstack security group list | grep $(openstack project show admin -f value -c id) | awk '{ print $2 }')
openstack security group rule create --proto icmp $sg_id
openstack security group rule create --dst-port 22 --protocol tcp $sg_id
openstack security group rule create --dst-port 80 --protocol tcp $sg_id

openstack server create --image cirros --flavor m1.tiny --nic net-id=provider,v4-fixed-ip=192.168.60.100 vm1
```

Task 2 
```
openstack network create private --dns-domain example.com.
openstack subnet create sub_private --network private --subnet-range 192.168.100.0/24 --dns-nameserver 8.8.8.8

openstack network create public  --share --external --provider-physical-network datacentre --provider-network-type vlan --provider-segment 10

openstack subnet create public --no-dhcp --network public --subnet-range 10.0.0.0/24   --allocation-pool start=10.0.0.71,end=10.0.0.200 --gateway 10.0.0.1 --dns-nameserver 8.8.8.8

openstack router create router_private

openstack router set router_private --external-gateway public

openstack router add subnet router_private sub_private


cat > webserver.sh << 'EOF'
#!/bin/sh

MYIP=$(/sbin/ifconfig eth0|grep 'inet addr'|awk -F: '{print $2}'| awk '{print $1}');
OUTPUT_STR="Welcome to $MYIP\r"
OUTPUT_LEN=${#OUTPUT_STR}

while true; do
    echo -e "HTTP/1.0 200 OK\r\nContent-Length: ${OUTPUT_LEN}\r\n\r\n${OUTPUT_STR}" | sudo nc -l -p 80
done
EOF

openstack server create --image cirros --flavor m1.tiny --nic net-id=private --user-data webserver.sh web01  --wait
openstack server create --image cirros --flavor m1.tiny --nic net-id=private --user-data webserver.sh web02  --wait

openstack loadbalancer create --name lbweb --vip-subnet-id sub_private

openstack loadbalancer list

openstack loadbalancer listener create --name listenerweb --protocol HTTP --protocol-port 80 lbweb

openstack loadbalancer pool create --name poolweb --protocol HTTP  --listener listenerweb --lb-algorithm ROUND_ROBIN

IPWEB01=$(openstack server show web01 -c addresses -f value | cut -d"=" -f2)
IPWEB02=$(openstack server show web02 -c addresses -f value | cut -d"=" -f2)
openstack loadbalancer member create --name web01 --address $IPWEB01 --protocol-port 80 poolweb
openstack loadbalancer member create --name web02 --address $IPWEB02 --protocol-port 80 poolweb

VIP=$(openstack loadbalancer show lbweb -c vip_address -f value)
PORTID=$(openstack port list --fixed-ip ip-address=$VIP -c ID -f value)
openstack floating ip create --port $PORTID public

openstack loadbalancer amphora list

# Access the server using the floating IP addresses associated with the load balancer:

openstack floating ip create --floating-ip-address 10.0.0.70 public

openstack server add floating ip testmetadata 10.0.0.70
```

```

656d02f7796f8df952ea0e62f16530b6

    "name": "user_key",
    "in": "query",
    "description": "Your API access key",
    "required": true,
    "x-data-threescale-name": "user_keys",
    "type": "string"

            "name": "lang",
            "in": "query",
            "description": "language of labels, e.g. \"en\" or \"fi\"",
            "required": true,
            "type": "string"

https://api-2445583270468.staging.gw.apicast.io:443

curl -v https://api-2445583270468.production.gw.apicast.io:443/rest/v1/vocabularies?lang=en?user_key=21b7921ee48f976c301ce86d5b8feb6c


curl "https://api-2445583270468.production.gw.apicast.io:443/rest/v1/vocabularies?lang=en&user_key=21b7921ee48f976c301ce86d5b8feb6c"

curl "https://api-2445583270468.production.gw.apicast.io:443/rest/v1/vocabularies?lang=en&user_key=656d02f7796f8df952ea0e62f16530b6"


<div class="row">
  {% if provider.multiple_applications_allowed? %}
    {% if applications == present %}
      <div id="client-name" data-client="{{ applications.first.id }}" style="display:none"></div>
      # {% include 'stats/chart' %}
    {% else %}
      <p>You do not currently have any live applications.</p>
    {% endif %}
  {% else %}
    <div id="client-name" data-client="{{ applications.first.id }}" style="display:none"></div>
    {% include 'stats/chart' %}
  {% endif %}
</div>

```

```
/opt/install/amq-broker-7.6.0/bin/artemis create  --user amquser --password amquser --role amq --allow-anonymous /opt/install/amq7/broker0

cat /opt/install/amq7/broker0/etc/broker.xml | grep journal-type

sed -i 's/localhost/'$HOSTNAME'/' /opt/install/amq7/broker0/etc/bootstrap.xml

cat /opt/install/amq7/broker0/etc/jolokia-access.xml
sed -i 's#<strict-checking/>#<!-- strict-checking -->#'  /opt/install/amq7/broker0/etc/jolokia-access.xml
cat /opt/install/amq7/broker0/etc/jolokia-access.xml

/opt/install/amq7/broker0/bin/artemis-service start

ps -aef | grep java

sudo netstat -ntuapee | grep LISTEN | grep java
sudo netstat -antup | grep LISTEN | grep java

# view broker log
cat /opt/install/amq7/broker0/log/artemis.log

# create a durable messaging address and queue
/opt/install/amq7/broker0/bin/artemis address create --name gpteAddress --anycast --no-multicast

# Associate a durable anycast queue with the previously created address
/opt/install/amq7/broker0/bin/artemis queue create --name gpteQueue --address gpteAddress --anycast --durable --auto-create-address

# consume messages from gpteQueue
/opt/install/amq7/broker0/bin/artemis consumer --destination gpteQueue --url tcp://$HOSTNAME:61616

# send a total of 20 10-byte messages via two parallel threads with a one-second sleep between messages
/opt/install/amq7/broker0/bin/artemis producer --destination gpteAddress --message-count 10 --message-size 10 --sleep 1000 --threads 2 --url tcp://$HOSTNAME:61616

# Stop the broker0 instance
/opt/install/amq7/broker0/bin/artemis-service stop

# Delete everything about the broker0 instance
rm -rf /opt/install/amq7/broker0/
```

```
# AMQP Lab
# Set Up Lab Assets
## change to the /opt directory, make a router-dev subdirectory, set permissions on this directory, and change to it:
cd /opt
sudo mkdir router-dev
sudo chown -R <your user>:users /opt/router-dev
cd router-dev

## Clone the amqp-dispatch-workshop lab assets
git clone https://github.com/gpe-mw-training/amqp-dispatch-workshop

# Use AMQP Wire Communication
## Create AMQP Broker
cd /opt/install/amq7

## stop and remove any older AMQ brokers
"/opt/install/amq7/broker0/bin/artemis-service" stop
rm -rf /opt/install/amq7/broker0

## Create an instance of the AMQ broker called broker0 and specify the credentials for a guest user and role
/opt/install/amq-broker-7.6.0/bin/artemis create  --user amquser --password amquser --role amq --allow-anonymous /opt/install/amq7/broker0

## Start Broker Instance
## replace localhost with $HOSTNAME in the /opt/install/amq7/broker0/etc/bootstrap.xml file
echo $HOSTNAME
sed -i 's/localhost/'$HOSTNAME'/' /opt/install/amq7/broker0/etc/bootstrap.xml

## In the /opt/install/amq7/broker0/etc/jolokia-access.xml file, comment out the following line to disable strict checking
cat /opt/install/amq7/broker0/etc/jolokia-access.xml
sed -i 's#<strict-checking/>#<!-- strict-checking -->#'  /opt/install/amq7/broker0/etc/jolokia-access.xml
cat /opt/install/amq7/broker0/etc/jolokia-access.xml

## Start the broker instance
"/opt/install/amq7/broker0/bin/artemis-service" start

## Create Durable Queue
## Create an anycast address called gpteAddress
/opt/install/amq7/broker0/bin/artemis address create --name gpteAddress --anycast --no-multicast

## Associate a durable anycast queue with the previously created address
/opt/install/amq7/broker0/bin/artemis queue create \
                              --name gpteQueue \
                              --address gpteAddress \
                              --anycast --durable \
                              --auto-create-address

## Send and Consume JMS Messages Using AMQP
## Change to the simple-jms-request-response directory of your lab assets
cd /opt/router-dev/amqp-dispatch-workshop/simple-jms-request-response/

## Review the com.redhat.gpte.amqp.examples.simplejms.AMQPQueueExample class

## At the root of the project, review the pom.xml Maven build file

## From the project root, identify the exact dependency tree that includes the supported qpid libraries
mvn dependency:tree

## Use a JMS client to produce and consume a message using AMQP from the root of the simple-jms-request-response project
mvn clean package exec:java \
          -Dexec.mainClass="org.apache.activemq.artemis.jms.example.AMQPQueueExample" \
          -DQUEUE_NAME="gpteAddress" \
          -DCONNECTION_URL="amqp://0.0.0.0:5672" \
          -Dexec.cleanupDaemonThreads=false 2>&1 | tee /tmp/err 

## Set Up qdrouterd Dispatch Router
## Verify Installation
sudo yum list qpid-\* | more

## Configure Dispatch Router
## Make a backup of the qpid-dispatch configuration file
sudo cp /etc/qpid-dispatch/qdrouterd.conf /etc/qpid-dispatch/qdrouterd.conf.bk

## make small change to /etc/qpid-dispatch/qdrouterd.conf
sudo yum install -y ansible

ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''

cat > ~/inventory << 'EOF'
[localhost]
localhost ansible_host=127.0.0.1 
EOF

ansible -i ~/inventory localhost -m authorized_key -a 'user=jwang-redhat.com state=present key="{{ lookup(\"file\",\"/home/jwang-redhat.com/.ssh/id_rsa.pub\") }}"'

cat > ~/.ssh/config << 'EOF'
Host *
  StrictHostKeyChecking no
EOF

cat > playbook.yml << 'EOF'
- name: playbook for qpid-dispatch qdrouterd
  hosts: localhost
  become: true

  tasks:
  - name: Insert/Update router configuration stanza in /etc/qpid-dispatch/qdrouterd.conf
    blockinfile:
      path: /etc/qpid-dispatch/qdrouterd.conf
      backup: yes
      insertafter: "mode: standalone"
      block: |4
              id: gpteRoute
              workerThreads: 4
EOF

ansible-playbook -i ~/inventory playbook.yml

## Save your changes to /etc/qpid-dispatch/qdrouterd.conf
sudo cat > /etc/qpid-dispatch/qdrouterd.conf << 'EOF'
router {
    mode: standalone
    id: gpteRoute
    workerThreads: 4
}

listener {
    host: 0.0.0.0
    port: 2009
    authenticatePeer: no
    role: normal
    saslMechanisms: ANONYMOUS
}

connector {
    name: brokerConnector
    host: localhost
    port: 5672
    role: route-container
    allowRedirect: no
}

address {
    prefix: gpteAddress
    waypoint: yes
}

autoLink {
    addr: gpteAddress
    dir: in
    connection: brokerConnector
}

autoLink {
    addr: gpteAddress
    dir: out
    connection: brokerConnector
}
EOF

## Start the qpid-dispatch router
sudo systemctl restart qdrouterd.service

## Verify that the dispatch-router is listening on the desired socket (0.0.0.0:2009)
sudo netstat -ntuapee | grep LISTEN | grep 2009

## View the connection status and statistics of the running qdrouterd service
qdstat -b 0.0.0.0:2009 -c

## View and tail the latest logs from the qdrouterd service
sudo journalctl -u qdrouterd -n 200 -f

# Publish and Subscribe JMS messages Using AMQP
## From anywhere on the file system of your lab VM, start the websockify proxy
websockify 0.0.0.0:5673 0.0.0.0:2009 &

## Send and Consume JMS Messages Using AMQP
## change to the simple-jms-request-response project directory
cd /opt/router-dev/amqp-dispatch-workshop/simple-jms-request-response

## Produce and consume a single JMS message by communicating via AMQP to the dispatch-router that listens on 0.0.0.0:2009
mvn clean package exec:java \
          -Dexec.mainClass="org.apache.activemq.artemis.jms.example.AMQPQueueExample" \
          -DQUEUE_NAME="gpteAddress" \
          -DCONNECTION_URL="amqp://$HOSTNAME:2009" \
          -Dexec.cleanupDaemonThreads=false
```

```
# Router Network Pattern Lab
# Review router-network Project
cd /opt/router-dev/amqp-dispatch-workshop/router-network

# start router network
./start.sh

# View running containers
podman ps -a

# Make sure that router1 is bound to port 2009
sudo netstat -ntuapee | grep LISTEN | grep 2009

# Use AMQP Clients
# start qpid client receiver
IP=$(ip addr | grep 'state UP' -A2 | tail -n1 | awk '{print $2}' | cut -f1  -d'/')
podman run -e QPID_LOG_ENABLE=trace+  --add-host=router4:$IP -t -i scholzj/qpid-cpp:latest qpid-receive -b router4:5672 --connection-options "{protocol: amqp1.0}" -a "'/myAddress'" -m 1 -f --print-headers yes

# start qpid client sender
IP=$(ip addr | grep 'state UP' -A2 | tail -n1 | awk '{print $2}' | cut -f1  -d'/')
podman run -e QPID_LOG_ENABLE=trace --add-host=router1:$IP -t -i scholzj/qpid-cpp:latest qpid-send -b router1:5672 --connection-options "{protocol: amqp1.0}" -a "'/myAddress'" -m 1
```

```
psql -Usampledb
sampledb-> \d
sampledb-> \x
sampledb-> \df+ add_lead
sampledb=> \df+ add_lead                                                                                                                 
List of functions                                                                                                                        
-[ RECORD 1 ]-------+--------------------------------------------------------------------------------------------------------------------
------------------------------------------------------------------------------------------------------------                             
Schema              | sampledb                                                                                                           
Name                | add_lead                                                                                                           
Result data type    | void                                                                                                               
Argument data types | first_and_last_name character varying, company character varying, phone character varying, email character varying,
 lead_source character varying, lead_status character varying, rating character varying                                                  
Type                | normal                                                                                                             
Volatility          | volatile
Parallel            | unsafe
Owner               | sampledb
Security            | invoker
Access privileges   | 
Language            | plpgsql
Source code         |             
                                                                                                            +
                    |   DECLARE   
                                                                                                            +
                    |     task varchar;                             
                                                                                                            +
                    |   BEGIN     
                                                                                                            +
                    |     task := concat(lead_status || ' ', 'Lead: Please contact ', first_and_last_name, ' from ' || company, ' via pho
ne: ' || phone, ' via email: ' || email, '. ', 'Lead is from ' || lead_source, '. Rating: ' || rating, '.');+
                    |     insert into todo(task,completed) VALUES (task,0);                                                              
                                                                                                            +
                    |   END;      
                                                                                                            +
                                                                                                            +
                    |   
Description         | 
sampledb=> select * from todo ;     
-[ RECORD 1 ]-------------------------------------------------------------------------------------------------------------
id        | 1
task      | send email to jwang@redhat.com
completed | 0
-[ RECORD 2 ]-------------------------------------------------------------------------------------------------------------
id        | 2
task      | Open - Not Contacted Lead: Please contact Wang from Noexist via phone: 60000000 via email: a@aaa.com. .
completed | 0
-[ RECORD 3 ]-------------------------------------------------------------------------------------------------------------
id        | 3
task      | Open - Not Contacted Lead: Please contact Pang from Noexist via phone: 60000000 via email: b@aaa.com. .
completed | 0
```

```
# steps for install apache maven
http://maven.apache.org/install.html

# Install Maven for Fuse
https://access.redhat.com/documentation/en-us/red_hat_fuse/7.5/html/installing_on_jboss_eap/set-up-maven-locally

# Install SoapUI
https://www.soapui.org/downloads/soapui/source-forge.html
```

```
# rhel8
nmcli con mod 'ens3' ipv4.method 'manual' ipv4.address '10.66.208.130/24' ipv4.gateway '10.66.208.254' ipv4.dns '10.64.63.6'

# rhel7
nmcli con mod 'eth0' ipv4.method 'manual' ipv4.address '10.66.208.130/24' ipv4.gateway '10.66.208.254' ipv4.dns '10.64.63.6'

# disconnected rhel8
nmcli con mod 'ens3' ipv4.method 'manual' ipv4.address '10.66.208.130/24' ipv4.gateway '10.66.208.254'

# disconnected rhel7
nmcli con mod 'eth0' ipv4.method 'manual' ipv4.address '10.66.208.130/24' ipv4.gateway '10.66.208.254'

# rhel8
cat > /etc/yum.repos.d/public.repo << 'EOF'
[rhel-8-for-x86_64-baseos-rpms]
name=rhel-8-for-x86_64-baseos-rpms
baseurl=http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-baseos-rpms/
gpgcheck=0
enabled=1

[rhel-8-for-x86_64-appstream-rpms]
name=rhel-8-for-x86_64-appstream-rpms
baseurl=http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-appstream-rpms/
gpgcheck=0
enabled=1

[ansible-2.8-for-rhel-8-x86_64-rpms]
name=ansible-2.8-for-rhel-8-x86_64-rpms
baseurl=http://10.66.208.158/rhel8osp/ansible-2.8-for-rhel-8-x86_64-rpms/
gpgcheck=0
enabled=1
EOF

# rhel7
cat > /etc/yum.repos.d/public.repo << 'EOF'
[rhel-7-server-rpms]
name=rhel-7-server-rpms
baseurl=http://10.66.208.115/rhel7osp/rhel-7-server-rpms/
gpgcheck=0
enabled=1

[rhel-server-rhscl-7-rpms]
name=rhel-server-rhscl-7-rpms
baseurl=http://10.66.208.115/rhel7osp/rhel-server-rhscl-7-rpms/
gpgcheck=0
enabled=1
EOF

# rhel7 (optional)
yum install -y dnf
dnf update -y

curl -O https://releases.ansible.com/ansible-tower/setup-bundle/ansible-tower-setup-bundle-latest.el7.tar.gz
tar zxvf ansible-tower-setup*.tar.gz
cd ansible-tower-setup*

# rhel7
cat > inventory << 'EOF'
[tower]
localhost ansible_connection=local

[database]
localhost ansible_connection=local

[all:vars]
admin_password='redhat'

pg_host='localhost'
pg_port='5432'

pg_database='awx'
pg_username='awx'
pg_password='redhat'

rabbitmq_username=tower
rabbitmq_password='redhat'
rabbitmq_cookie=cookiemonster
EOF

# rhel 8
cat > inventory << 'EOF'
[tower]
localhost ansible_connection=local

[database]
localhost ansible_connection=local

[all:vars]
admin_password='redhat'

pg_host='localhost'
pg_port='5432'

pg_database='awx'
pg_username='awx'
pg_password='redhat'

rabbitmq_username=tower
rabbitmq_password='redhat'
rabbitmq_cookie=cookiemonster

nginx_http_port='80'
nginx_https_port='443'
EOF

# rhel7
mkdir -p /repos/ansible-tower-dependencies
pushd /repos/ansible-tower-dependencies
wget \
     --recursive \
     --no-clobber \
     --page-requisites \
     --html-extension \
     --convert-links \
     --domains ansible.com \
     --no-parent \
         https://releases.ansible.com/ansible-tower/rpm/dependencies/3.7/epel-7-x86_64/

# rhel8
mkdir -p /repos/ansible-tower-dependencies
pushd /repos/ansible-tower-dependencies
wget \
     --recursive \
     --no-clobber \
     --page-requisites \
     --html-extension \
     --convert-links \
     --domains ansible.com \
     --no-parent \
         https://releases.ansible.com/ansible-tower/rpm/dependencies/3.7/epel-8-x86_64/

# cleanup
mv releases.ansible.com/ansible-tower/rpm/dependencies/3.7/epel-7-x86_64/*.rpm . 
mv releases.ansible.com/ansible-tower/rpm/dependencies/3.7/epel-7-x86_64/repodata/ .
rm -rf releases.ansible.com/ 

dnf install -y ansible

# rhel7
mkdir -p /repos/ansible-tower
pushd /repos/ansible-tower
wget https://releases.ansible.com/ansible-tower/rpm/epel-7-x86_64/ansible-tower-3.7.2-1.el7at.x86_64.rpm
wget https://releases.ansible.com/ansible-tower/rpm/epel-7-x86_64/ansible-tower-cli-3.7.2-1.el7at.x86_64.rpm
wget https://releases.ansible.com/ansible-tower/rpm/epel-7-x86_64/ansible-tower-isolated-3.7.2-1.el7at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-7-x86_64/ansible-tower-server-3.7.2-1.el7at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-7-x86_64/ansible-tower-setup-3.7.2-1.el7at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-7-x86_64/ansible-tower-ui-3.7.2-1.el7at.x86_64.rpm
wget https://releases.ansible.com/ansible-tower/rpm/epel-7-x86_64/ansible-tower-venv-ansible-3.7.2-1.el7at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-7-x86_64/ansible-tower-venv-tower-3.7.2-1.el7at.x86_64.rpm

# rhel8
mkdir -p /repos/ansible-tower
pushd /repos/ansible-tower
wget https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/ansible-tower-3.7.2-1.el8at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/ansible-tower-cli-3.7.2-1.el8at.x86_64.rpm
wget https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/ansible-tower-isolated-3.7.2-1.el8at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/ansible-tower-server-3.7.2-1.el8at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/ansible-tower-setup-3.7.2-1.el8at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/ansible-tower-ui-3.7.2-1.el8at.x86_64.rpm
wget https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/ansible-tower-venv-ansible-3.7.2-1.el8at.x86_64.rpm 
wget https://releases.ansible.com/ansible-tower/rpm/epel-8-x86_64/ansible-tower-venv-tower-3.7.2-1.el8at.x86_64.rpm 

createrepo .

popd
popd


# rhel7 and rhel8
cat > /root/ansible-tower.repo << 'EOF'
[ansible-tower]
name=Ansible Tower Repository - $releasever $basearch
baseurl=file:///repos/ansible-tower
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[ansible-tower-dependencies]
name=Ansible Tower Dependencies Repository - $releasever $basearch
baseurl=file:///repos/ansible-tower-dependencies
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

dnf repolist --all

# rhel7 and rhel8
cat /root/test.sh << 'EOF'
for ((;;))
do
  sleep .1
  echo yes | cp -rf /root/ansible-tower.repo /etc/yum.repos.d
done
EOF

# on screen session 1, run command: /bin/bash -x /root/test.sh

```

```
# 从以下网址下载 ansible-tower-3.7.2 bundle rhel8 离线 repo 
https://drive.google.com/file/d/155jVxypKKPg_0zw2Rs3A0Aq7pQzwik9D/view?usp=sharing

# 在目标主机上准备 rhel8 yum repo rhel-8-for-x86_64-baseos-rpms, rhel-8-for-x86_64-appstream-rpms 和 ansible-2.8-for-rhel-8-x86_64-rpms
cat > /etc/yum.repos.d/public.repo << 'EOF'
[rhel-8-for-x86_64-baseos-rpms]
name=rhel-8-for-x86_64-baseos-rpms
baseurl=http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-baseos-rpms/
gpgcheck=0
enabled=1

[rhel-8-for-x86_64-appstream-rpms]
name=rhel-8-for-x86_64-appstream-rpms
baseurl=http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-appstream-rpms/
gpgcheck=0
enabled=1

[ansible-2.8-for-rhel-8-x86_64-rpms]
name=ansible-2.8-for-rhel-8-x86_64-rpms
baseurl=http://10.66.208.158/rhel8osp/ansible-2.8-for-rhel-8-x86_64-rpms/
gpgcheck=0
enabled=1
EOF

# 在目标主机上准备本地文件，用于覆盖 ansible-tower 安装程序生成的 repo 文件
cat > /root/ansible-tower.repo << 'EOF'
[ansible-tower]
name=Ansible Tower Repository - $releasever $basearch
baseurl=file:///repos/ansible-tower
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[ansible-tower-dependencies]
name=Ansible Tower Dependencies Repository - $releasever $basearch
baseurl=file:///repos/ansible-tower-dependencies
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

# 在目标主机上生成后台刷新脚本
cat /root/test.sh << 'EOF'
for ((;;))
do
  sleep .1
  echo yes | cp -rf /root/ansible-tower.repo /etc/yum.repos.d
done
EOF

# 在目标主机上的1个终端里运行此脚本
/bin/bash -x /root/test.sh

# : ./setup 
```

```
# disconnected rhel7 tower-support-01
hostnamectl set-hostname jwang-tower-support-01.example.com

cat >> /etc/hosts << 'EOF'
10.66.208.162 jwang-tower-01.example.com
10.66.208.163 jwang-tower-db-01.example.com
EOF
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa

# disconnected rhel7 tower-01
hostnamectl set-hostname jwang-tower-01.example.com
nmcli con mod 'eth0' ipv4.method 'manual' ipv4.address '10.66.208.162/24' ipv4.gateway '10.66.208.254'
cat >> /etc/hosts << 'EOF'
10.66.208.162 jwang-tower-01.example.com
10.66.208.163 jwang-tower-db-01.example.com
EOF

# disconnected rhel7 tower-db-01
hostnamectl set-hostname jwang-tower-db-01.example.com
nmcli con mod 'eth0' ipv4.method 'manual' ipv4.address '10.66.208.163/24' ipv4.gateway '10.66.208.254'
cat >> /etc/hosts << 'EOF'
10.66.208.162 jwang-tower-01.example.com
10.66.208.163 jwang-tower-db-01.example.com
EOF


cat > inventory << 'EOF'
[tower]
jwang-tower-01.example.com

[database]
jwang-tower-db-01.example.com

[all:vars]
admin_password='redhat'

pg_host='jwang-tower-db-01.example.com'
pg_port='5432'

pg_database='awx'
pg_username='awx'
pg_password='redhat'
EOF
```