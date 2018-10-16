

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
ovirt-node: centos162, 192.168.122.162, 192.168.130.162, default, 192_168.130_0  
ovirt-node: centos163, 192.168.122.163, 192.168.130.163, default, 192_168.130_0  
contrail-controller: centos164, 192.168.122.164, 192.168.130.164: default, 192_168.130_0
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
    ntpserver: ntp.nict.jp
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

Since libvirt tcp port is used by vdsm, nova_libvirt has to be stopped.
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


