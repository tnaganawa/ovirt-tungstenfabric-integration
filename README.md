

# version
ovirt 4.2.6.4-1.el7  
tungsten-fabric r5.0.1  
centos7.5


# setup

## oVirt installation document
https://ovirt.org/documentation/quickstart/quickstart-guide/

## all the vms are served by centos7 libvirt

## libvirt-network
default, 192.168.122.0/24, nat  
192_168_130_0, 192.168.130.0/24, isolated


## nodes
ovirt-manager: centos161, 192.168.122.161 default  
ovirt-node: centos162, 192.168.122.162, 192.168.130.162, default, 192_168_130_0  
ovirt-node: centos163, 192.168.122.163, 192.168.130.163, default, 192_168_130_0  
contrail-controller: centos164, 192.168.122.164, 192.168.130.164: default, 192_168_130_0
- 4vcpu, 8GB mem (24GB for contrail-controller), 48GB disk

# oVirt setting

```
          --== CONFIGURATION PREVIEW ==--

          Application mode                        : both
          Default SAN wipe after delete           : False
          Firewall manager                        : firewalld
          Update Firewall                         : True
          Host FQDN                               : centos161
          Configure local Engine database         : True
          Set application as default page         : True
          Configure Apache SSL                    : True
          Engine database secured connection      : False
          Engine database user name               : engine
          Engine database name                    : engine
          Engine database host                    : localhost
          Engine database port                    : 5432
          Engine database host name validation    : False
          Engine installation                     : True
          PKI organization                        : Test
          Set up ovirt-provider-ovn               : True
          Configure WebSocket Proxy               : True
          DWH installation                        : True
          DWH database host                       : localhost
          DWH database port                       : 5432
          Configure local DWH database            : True
          Configure Image I/O Proxy               : True
          Configure VMConsole Proxy               : True

          Please confirm installation settings (OK, Cancel) [OK]:
```


# node setting

## Add two hosts and one NFS data domain

cpu-type 'cpu-model' has to be specified

```
# virsh edit centos162
# virsh edit centos163
```

```
<cpu mode='host-model'>
</cpu>
```


## Import cirros image from ovirt glance

## create cirros vms on two hosts and check if vms work well


# Install tungsten-fabric

Use tungstenfabric r5.0.1  
https://hub.docker.com/u/tungstenfabric/

and ansible-deployer to install openstack neutron / keystone  
https://github.com/Juniper/contrail-ansible-deployer/wiki/Contrail-with-Openstack-Kolla

- instance.yaml

```
provider_config:
  bms:
    ssh_pwd: root
    ssh_user: root
    domainsuffix: local
    ntpserver: 0.centos.pool.ntp.org
instances:
  bms1:
    provider: bms
    ip: 192.168.122.164
    roles:
      config_database:
      config:
      control:
      analytics_database:
      analytics:
      webui:
      openstack:
  bms11:
    provider: bms
    ip: 192.168.122.162
    roles:
      vrouter:
        PHYSICAL_INTERFACE: eth1
        VROUTER_GATEWAY: 192.168.130.1
      openstack_compute:
  bms12:
    provider: bms
    ip: 192.168.122.163
    roles:
      vrouter:
        PHYSICAL_INTERFACE: eth1
        VROUTER_GATEWAY: 192.168.130.1
      openstack_compute:
contrail_configuration:
  RABBITMQ_NODE_PORT: 5673
  AUTH_MODE: keystone
  KEYSTONE_AUTH_URL_VERSION: /v3
  CONTRAIL_VERSION: r5.0.1
kolla_config:
  kolla_globals:
    enable_haproxy: no
    enable_swift: no
  kolla_passwords:
    keystone_admin_password: contrail123
global_configuration:
  CONTAINER_REGISTRY: tungstenfabric
```


For ovirt-nodes, 'vrouter' and 'nova_compute' role have to be specified  
PHYSICAL_INTERFACE, VROUTER_GATEWAY also need to be explicitly specified to make vrouter use eth1 (not the same NIC with ovirtmgmt)



## stop nova_compute, nova_libvirt 

Since libvirt port is used by vdsm, nova_libvirt need to be stopped.
```
# docker stop nova_compute nova_libvirt
# docker rm nova_compute nova_libvirt
# reboot
```


# Create openstack virtual-network

type this command to create virtual-network on tungsten fabric
```
# source /etc/kolla/kolla-toolbox/admin-openrc.sh
# openstack network create vn1
# openstack subnet create --subnet-range 10.0.1.0/24 --network vn1 subnet1
```


# Setup oVirt neutron provider

Create tungsten-fabric provider with this parameter.

```
Name: tungsten-fabric
Network Plugin: LINUX_BRIDGE
Provider URL: http://192.168.122.164:9696
Read-Only: Checked
Requires Authentication: Checked
Username: admin
Password: contrail123
Tenant /name: admin
Authentication URL: http://192.168.122.164:35357/v2.0
```


# Import virtual-network from tungsten-fabric


# add vdsm hook

locate this file on /usr/libexec/vdsm/hooks/after_device_create/60_tungsten-fabric
- tf_controller_ip have to be given when deployed

```
#!/usr/bin/python
import os
import requests

tf_controller_ip='192.168.122.164'
vnic_id=os.environ['vnic_id']

re = requests.get('http://{}:8082/virtual-machine-interface/{}'.format (tf_controller_ip, vnic_id))
js = re.json()['virtual-machine-interface']
mac_addr =  (js['virtual_machine_interface_mac_addresses']['mac_address'][0])
vm_id = (js['virtual_machine_refs'][0]['to'][0])
vn_id = (js['virtual_network_refs'][0]['uuid'])

cmd = "/var/lib/docker/volumes/opt_plugin/_data/bin/vrouter-port-control --oper=add --uuid={} --instance_uuid={} --vn_uuid={} --vm_name='' --ip_address='0.0.0.0' --ipv6_address=None --tap_name=tap{} --mac='{}' --rx_vlan_id=-1 --tx_vlan_id=-1".format(vnic_id, vm_id, vn_id, vnic_id[:11], mac_addr)

os.popen(cmd).read()
```


```
# chmod 755 /usr/libexec/vdsm/hooks/after_device_create/60_tungsten-fabric
# chmod 755 /var/lib/docker/volumes/
```


# Create vm and attach vNIC on TF virtual-network


## login to cirros and check ping works well

```
# ping 10.0.1.1 # gw-ip  
# ping 10.0.1.2 # service-ip  
# ping (cirros ip on different ovirt node)
```


```
[root@centos163 ~]# ip -o a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lftforever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
21: ovirtmgmt    inet 192.168.122.163/24 brd 192.168.122.255 scope global ovirtmgmt\       valid_lft forever preferred_lft forever
21: ovirtmgmt    inet6 fe80::5054:ff:fe7b:b7ac/64 scope link \       valid_lftforever preferred_lft forever
25: vhost0    inet 192.168.130.163/24 brd 192.168.130.255 scope global vhost0\      valid_lft forever preferred_lft forever
25: vhost0    inet6 fe80::5054:ff:fecf:54be/64 scope link \       valid_lft forever preferred_lft forever
26: docker0    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0\      valid_lft forever preferred_lft forever
27: genev_sys_6081    inet6 fe80::c8b:b5ff:fe21:bbed/64 scope link \       valid_lft forever preferred_lft forever
28: pkt0    inet6 fe80::f806:bcff:fe7d:fe28/64 scope link \       valid_lft forever preferred_lft forever
30: tapec01038d-44    inet6 fe80::fc1a:4aff:fe16:105/64 scope link \       valid_lft forever preferred_lft forever
[root@centos163 ~]#
[root@centos163 ~]# ip route
default via 192.168.122.1 dev ovirtmgmt
169.254.0.0/16 dev ovirtmgmt scope link metric 1021
169.254.0.3 dev vhost0 proto 109 scope link
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.122.0/24 dev ovirtmgmt proto kernel scope link src 192.168.122.163
192.168.130.0/24 dev vhost0 proto kernel scope link src 192.168.130.163
[root@centos163 ~]#
[root@centos163 ~]# ssh cirros@169.254.0.3
The authenticity of host '169.254.0.3 (169.254.0.3)' can't be established.
ECDSA key fingerprint is SHA256:HVJoTV0MGH9/T8bIw0aofzX7rCAphKDgts36YAXxpoo.
ECDSA key fingerprint is MD5:03:55:f1:dd:53:ed:c9:87:62:fd:e6:3a:bb:59:aa:cc.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '169.254.0.3' (ECDSA) to the list of known hosts.
cirros@169.254.0.3's password:
$
$ ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000\    link/ether 00:1a:4a:16:01:05 brd ff:ff:ff:ff:ff:ff
2: eth0    inet 10.0.1.4/24 brd 10.0.1.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::21a:4aff:fe16:105/64 scope link \       valid_lft forever preferred_lft forever
$
$ ping 10.0.1.3
PING 10.0.1.3 (10.0.1.3): 56 data bytes
64 bytes from 10.0.1.3: seq=0 ttl=64 time=2.855 ms
64 bytes from 10.0.1.3: seq=1 ttl=64 time=1.852 ms
^C
--- 10.0.1.3 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 1.852/2.353/2.855 ms
$
$ ssh 10.0.1.3

Host '10.0.1.3' is not in the trusted hosts file.
(ecdsa-sha2-nistp521 fingerprint md5 5f:61:d0:f8:c3:c2:aa:8d:07:95:29:b4:52:aa:06:77)
Do you want to continue connecting? (y/n) y
cirros@10.0.1.3's password:
$
$ ip -o a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1\    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
1: lo    inet6 ::1/128 scope host \       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000\    link/ether 00:1a:4a:16:01:04 brd ff:ff:ff:ff:ff:ff
2: eth0    inet 10.0.1.3/24 brd 10.0.1.255 scope global eth0\       valid_lft forever preferred_lft forever
2: eth0    inet6 fe80::21a:4aff:fe16:104/64 scope link \       valid_lft forever preferred_lft forever
$
```
