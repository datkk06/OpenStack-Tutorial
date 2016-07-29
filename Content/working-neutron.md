###Tenant Netwoks Scenario
The basic Neutron configuration uses tenant networks to provide internet access to the instances through an external physical network. All traffic coming from/to the Compute nodes is routed through the Network node that is attached on the external network. In this sections we are going to implement the basic scenario by configuring the tenant and the external networks.

####Configure the external network
In the basic networking scenario with a single flat external network, only administrative users can manage external networks because they use the physical network infrastructure.

On the Control node, login as ``admin`` user and create the external network
```
# source keystonerc_admin
# neutron net-create external-flat-network \
--shared \
--provider:network_type flat \
--provider:physical_network physnet \
--router:external True

Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 6ede0952-25f7-489d-9ce4-0126da7cb7d0 |
| mtu                       | 0                                    |
| name                      | external-flat-network                |
| provider:network_type     | flat                                 |
| provider:physical_network | physnet                              |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 5ccf7027366442709bde78831da6cce2     |
+---------------------------+--------------------------------------+
```

Pay attention to the following parameters:

* ``provider:network_type flat``
* ``provider:physical_network physnet``
* ``router:external True``
* ``shared``

The external network shares the same subnet and gateway associated with the physical network connected to the external interface on the Network node. Specify an exclusive slice of this subnet for router and IP addresses to prevent interference with other devices on the same external network

```
# neutron subnet-create external-flat-network 172.120.1.0/24  \
--name external-flat-subnetwork \
--gateway 172.120.1.1 \
--disable-dhcp \
--allocation-pool start=172.120.1.200,end=172.120.1.220

Created a new subnet:
+-------------------+---------------------------------------------------+
| Field             | Value                                             |
+-------------------+---------------------------------------------------+
| allocation_pools  | {"start": "172.120.1.200", "end": "172.120.1.220"}|
| cidr              | 172.120.1.0/24                                    |
| dns_nameservers   |                                                   |
| enable_dhcp       | False                                             |
| gateway_ip        | 172.120.1.1                                       |
| host_routes       |                                                   |
| id                | 40f89cb3-9474-48e0-ab4c-7fa3fb57009e              |
| ip_version        | 4                                                 |
| ipv6_address_mode |                                                   |
| ipv6_ra_mode      |                                                   |
| name              | external-flat-subnetwork                          |
| network_id        | 6ede0952-25f7-489d-9ce4-0126da7cb7d0              |
| subnetpool_id     |                                                   |
| tenant_id         | 5ccf7027366442709bde78831da6cce2                  |
+-------------------+---------------------------------------------------+
```

####Configure VxLAN encapsulation for Tenant networks
In this section, we are going to configure the tenant network encapsulation based on **VxLAN** as tunneling protocol. Other options are: **VLAN** and **GRE**. The **Virtual extensible LAN** tunnels realize a L2 network on top of IP routed interfaces. Each L2 frame is tagged with an appropriate **Virtual Network Identifier** and wrapped into UDP packets. Tunnel endpoints listen on 4789 port, by default. IP Multicast can be used to flood broadcast and unknown L2 traffic.

On the Control node, change the settings
```
# vi /etc/neutron/plugin.ini
[ml2]
type_drivers = flat, vxlan, vlan, gre
tenant_network_types = vxlan
mechanism_drivers = openvswitch, l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = physnet
# use flat_networks = * to allow flat networks with arbitrary names

[ml2_type_vxlan]
vni_ranges = 1001:2000
vxlan_group = 239.1.1.2

[ml2_type_vlan]
#network_vlan_ranges = physnet:1001:2000

[ml2_type_gre]
#tunnel_id_ranges = 1:1000

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_ipset = True
```
and restart the Neutron service
```
# systemctl restart neutron-server
```

On the Network node, change the settings
```
# vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
int_peer_patch_port = patch-tun
tun_peer_patch_port = patch-int
local_ip = LOCAL_TUNNEL_INTERFACE_IP_ADDRESS
bridge_mappings = physnet:br-ex

[agent]
enable_tunneling = True
tunnel_types = vxlan
vxlan_udp_port = 4789
enable_distributed_routing = False
l2_population = True
prevent_arp_spoofing = True

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

restart the OVS agent
```
# systemctl restart neutron-openvswitch-agent
```

and check the new OVS layout
```
# ovs-vsctl list-ports br-ex


# ovs-vsctl list-ports br-int


# ovs-vsctl list-ports br-tun

```

On all the Compute nodes, change the settings
```
# vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
int_peer_patch_port = patch-tun
tun_peer_patch_port = patch-int
local_ip = LOCAL_TUNNEL_INTERFACE_IP_ADDRESS
# uncomment when compute node is directly attached to external network
# bridge_mappings = physnet:br-ex

[agent]
enable_tunneling = True
tunnel_types = vxlan
vxlan_udp_port = 4789
enable_distributed_routing = False
l2_population = True
prevent_arp_spoofing = True

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

restart the OVS agent
```
# systemctl restart neutron-openvswitch-agent
```

and check the new OVS layout
```
# ovs-vsctl list-ports br-int


# ovs-vsctl list-ports br-tun

```

####Configure Tenant networks
The tenant networks are created by the tenant users. Login as a tenant user and create a tenant network
```
# source keystonerc_bcloud
# neutron net-create tenant-network
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | 72563bb0-b5ea-44d0-9a59-f95f7cc12671 |
| mtu             | 0                                    |
| name            | tenant-network                       |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | 22bdc5a0210e4a96add0cea90a6137ed     |
+-----------------+--------------------------------------+
```

Like the external network, a tenant network also requires a subnet attached to it. By default, this subnet will use DHCP so instances can obtain IP addresses.
```
# neutron subnet-create tenant-network 192.168.1.0/24 \
--name tenant-subnetwork \
--gateway 192.168.1.1 \
--enable-dhcp \
--dns-nameserver 8.8.8.8 \
--allocation-pool start=192.168.1.10,end=192.168.1.250

Created a new subnet:
+-------------------+---------------------------------------------------+
| Field             | Value                                             |
+-------------------+---------------------------------------------------+
| allocation_pools  | {"start": "192.168.1.10", "end": "192.168.1.250"} |
| cidr              | 192.168.1.0/24                                    |
| dns_nameservers   | 8.8.8.8                                           |
| enable_dhcp       | True                                              |
| gateway_ip        | 192.168.1.1                                       |
| host_routes       |                                                   |
| id                | 45f5980b-5a15-4372-93ab-b77502a6104a              |
| ip_version        | 4                                                 |
| ipv6_address_mode |                                                   |
| ipv6_ra_mode      |                                                   |
| name              | tenant-subnetwork                                 |
| network_id        | 72563bb0-b5ea-44d0-9a59-f95f7cc12671              |
| subnetpool_id     |                                                   |
| tenant_id         | 22bdc5a0210e4a96add0cea90a6137ed                  |
+-------------------+---------------------------------------------------+
```

Having enabled the DHCP, a DHCP server is created as dedicatd namespace. The DHCP server provides IP addresses to the virtual machine inside the internal network.
```
# ip netns
qdhcp-9a7f354c-7a46-420d-98a5-3508e6f3caf1
# ip netns exec qdhcp-9a7f354c-7a46-420d-98a5-3508e6f3caf1 ifconfig
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
tap9fc1c45b-f9: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.10  netmask 255.255.255.0  broadcast 192.168.1.255
```

The tenant network just created need to be connected to the external network via a virtual router. Create the virtual router as tenant user
```
# source keystonerc_demo
# neutron router-create mygateway
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | 074cf6a9-8299-4add-9bd6-69bbc6fd2f32 |
| name                  | mygateway                            |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 22bdc5a0210e4a96add0cea90a6137ed     |
+-----------------------+--------------------------------------+
```

As tenant user, create a router interface to connect tenant subnetwork with the external network
```
# neutron router-interface-add mygateway subnet=tenant-subnetwork
# neutron router-gateway-set mygateway external-flat-network
```

The virtual router just created lives in the Network node as private ip namespace. Check the network connectivity by the Network node
```
# ip netns
qrouter-9a45bdb6-b7ff-4329-8333-7339050ebcf9

# ip netns exec qrouter-9a45bdb6-b7ff-4329-8333-7339050ebcf9 ifconfig
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
qg-d62b9626-8b: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.1.200  netmask 255.255.255.0  broadcast 172.16.1.255
qr-9c0083e4-0a: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.1  netmask 255.255.255.0  broadcast 192.168.1.255
```

####Configure Security Groups
Security Groups control traffic incoming and outcoming to and from virtual machines. As tenant user, configure a securty group and add rules
```
# neutron security-group-create myaccess
# neutron security-group-list
+--------------------------------------+----------+----------------------+
| id                                   | name     | security_group_rules |
+--------------------------------------+----------+----------------------+
| 64fbc795-4c7f-4645-9565-7190ead608b4 | default  | egress, IPv4         |
| adc9c79d-7ac2-48bc-8572-116fba86b51e | myaccess | egress, IPv4         |
|                                      |          | egress, IPv6         |
+--------------------------------------+----------+----------------------+

# neutron security-group-rule-create \
--protocol icmp \
--direction ingress \
myaccess

# neutron security-group-rule-create \
--protocol tcp \
--port-range-min 22 \
--port-range-max 22 \
--direction ingress \
myaccess

# neutron security-group-rule-list
+--------------------------------------+----------------+-----------+-----------+---------------+--------+
| id                                   | security_group | direction | ethertype | protocol/port | remote |
+--------------------------------------+----------------+-----------+-----------+---------------+--------+
| 07dec775-7d3c-40b8-ab10-105b926224c9 | myaccess       | ingress   | IPv4      | 22/tcp        | any    |
| 48259cf9-f05e-481a-81b3-dd62a14386c5 | default        | egress    | IPv4      | any           | any    |
| 60b7cb14-8808-416d-bf68-eeb7fb3e3208 | myaccess       | egress    | IPv4      | any           | any    |
| 7ee00ccd-5792-4155-bcea-346b2130c537 | myaccess       | ingress   | IPv4      | icmp          | any    |
| a7db323f-87ce-4745-b0be-97f4d45f6ea3 | myaccess       | egress    | IPv6      | any           | any    |
+--------------------------------------+----------------+-----------+-----------+---------------+--------+
```

####Configure GRE Tunnel encapsulation for Tenant networks
In this section we are going to set the tunnel type used for the Tenant networks from the VxLAN to the **GRE** encapsulation. **Generic Routing Encapsulation** is a tunneling protocol (RFC2784) developed by **Cisco Systems** that can encapsulate a wide variety of network layer protocols inside virtual point-to-point links over an Internet Protocol network. In OpenStack, the GRE can be used as method to implement L2 Tenant networks over a L3 routed network.

On the Control node, change the settings
```
# vi /etc/neutron/plugin.ini
[ml2]
type_drivers = flat,vxlan,vlan,gre
tenant_network_types = gre
mechanism_drivers = openvswitch
...
[ml2_type_gre]
tunnel_id_ranges =1:1000
...
```
and restart the Neutron service
```
# systemctl restart neutron-server
```

On the Network node, change the settings
```
# vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
# In order to configure GRE Tunnel, all the physical NIC interfaces where GRE tunnel is attached must have an IP address.
# Also Multicast traffic should be permitted.
local_ip = 192.168.1.38
#
bridge_mappings = external:br-ex
enable_tunneling=True
...
[agent]
tunnel_types = gre
...
```

restart the OVS agent
```
# systemctl restart neutron-openvswitch-agent
```

and check the new OVS layout
```
# ovs-vsctl list-ports br-ex
ens33
phy-br-ex

# ovs-vsctl list-ports br-int
int-br-ex
patch-tun

# ovs-vsctl list-ports br-tun
gre-c0a80120
gre-c0a80122
patch-int
```

On all the Compute nodes, change the settings
```
# vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
# In order to configure GRE Tunnel, all the physical NIC interfaces where GRE tunnel is attached must have an IP address.
# Also Multicast traffic should be permitted.
local_ip = 192.168.1.32
#
enable_tunneling=True
...
[agent]
tunnel_types = gre
...
```

restart the OVS agent
```
# systemctl restart neutron-openvswitch-agent
```

and check the new OVS layout
```
# ovs-vsctl list-ports br-int
int-br-ex
patch-tun

# ovs-vsctl list-ports br-tun
gre-c0a80122
gre-c0a80126
patch-int
```

Create tenant and external networks as usual and start a new VM in order to check things happen as expected.

###Configure VLANs for Tenant networks
In this section we are going to use a VLAN L2 switch to implement the Tenant networks. The switch must support the VLAN trunking in order to get working.


