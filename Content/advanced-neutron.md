###Advanced Neutron Configuration
The basic Neutron configuration uses tenant networks to provide internet access to the instances. All traffic coming from the Compute nodes is routed through the Network node. In this sections we are going to enhance the basic scenario by introducing advanced features of the Neutron Service.

####Flat Provider network scenario
Provider networks are created by the OpenStack administrator and map directly to an existing physical network in the data center. Use flat provider networks to connect instances directly to the external network. 

On the Controller node, edit the ``/etc/neutron/plugin.ini`` initialization file

```
[ml2]
type_drivers = vxlan,flat
...
[ml2_type_flat]
flat_networks = *
...
```

Restart the Neutron service to apply the change
```
# systemctl restart neutron-server
```

On the Network node, configure the Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
bridge_mappings = external:br-ex
...
```

and restart the service
```
# systemctl restart neutron-openvswitch-agent
```

Configure the L3 Agent by editing the ``/etc/neutron/l3_agent.ini`` initialization file
```
[DEFAULT]
external_network_bridge =
...
```

Unlike the Tenant networks scenario where ``external_network_bridge = br-ex``, in the Provider networks scenario, Neutron L3 Agent uses a blank string for connecting the Network node to an external network. When this parameter is set to a blank string, Neutron allows multiple flat provider networks by creating a patch port from each bridge interface to the ``br-int`` bridge.

Restart neutron-l3-agent for the changes to take effect.
```
# systemctl restart neutron-l3-agent
```

On each Compute node, create an additional bridge interface to connect the nodes to the external network, and allow instances to communicate directly with the external network. For example, if we have a physical interface called ``ens36``, create an external network bridge ``br-ex`` and associate the it
```
# vi /etc/sysconfig/network-scripts/ifcfg-br-ex
DEVICE=br-ex
TYPE=OVSBridge
DEVICETYPE=ovs
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none

# vi /etc/sysconfig/network-scripts/ifcfg-ens36
DEVICE=ens36
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=none
```

Restart the network service to enable the bridge
```
# systemctl restart network
```

On each Compute node, configure the Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
bridge_mappings = external:br-ex
...
```

and restart the service
```
# systemctl restart neutron-openvswitch-agent
```

Now we are going to create an external network as a flat provider network and associate it with the configured physical network. Configuring it as a shared network will allow all tenants to create instances directly to it

```
# source keystone_admin
# neutron net-create public_net \
> --provider:network_type flat \
> --provider:physical_network external \
> --router:external=True \
> --shared

# neutron net-show public_net
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | c6d6e7cd-ba7d-4cfd-acb1-4cf37e2e4aaa |
| mtu                       | 0                                    |
| name                      | public_net                           |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 613b2bc016c5428397b3fea6dc162af1     |
+---------------------------+--------------------------------------+
```

Create the related subnet
```
# neutron subnet-create --name public_subnet \
> --enable_dhcp=True \
> --dns-nameserver=8.8.8.8 \
> --allocation_pool start=172.120.1.200,end=172.120.1.210 \
> --gateway=172.120.1.1
> public_net 172.120.1.0/24

Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "172.120.1.200", "end": "172.120.1.210"} |
| cidr              | 172.120.1.0/24                                     |
| dns_nameservers   | 8.8.8.8                                            |
| enable_dhcp       | True                                               |
| gateway_ip        | 172.120.1.1                                        |
| host_routes       |                                                    |
| id                | e52476da-5f0f-4606-8160-3324f58fcd5b               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | public_subnet                                      |
| network_id        | c6d6e7cd-ba7d-4cfd-acb1-4cf37e2e4aaa               |
| subnetpool_id     |                                                    |
| tenant_id         | 613b2bc016c5428397b3fea6dc162af1                   |
+-------------------+----------------------------------------------------+
```








####VLAN based Provider network scenario
