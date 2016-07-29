###Neutron Networking Service
OpenStack administrators can configure rich network topologies by creating and configuring networks and subnets. In particular, OpenStack networking supports each tenant having multiple private networks, and allows tenants to choose their own IP addressing space, even if those IP addresses overlap with those used by other tenants. This enables advanced cloud networking use cases, such as building multitiered web applications and allowing applications to be migrated to the cloud without changing IP addresses.

OpenStack networking uses the concept of a plug-in, which is a pluggable back-end implementation of the OpenStack networking API. A plug-in can use a variety of technologies. Some OpenStack networking plug-ins might use basic Linux networking, while others might use more advanced technologies, such as Open vSwitch, to provide similar benefits.

To enable OpenStack use Neutron for networking, on the Controller node, create the Keystone user and network service
```
# source /root/keystonerc_admin
# openstack-db --init --service neutron --password <password> --rootpw <password>
# keystone service-create --name neutron --type network --description 'Neutron Networking Service'
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |    Neutron Networking Service    |
|   enabled   |               True               |
|      id     | fe006cc01d234e93a308933d60c396f2 |
|     name    |             neutron              |
|     type    |             network              |
+-------------+----------------------------------+

# keystone endpoint-create \
--service network \
--publicurl http://controller:9696 \
--adminurl http://controller:9696 \
--internalurl http://controller:9696

# keystone user-create --name neutron --pass <password>
# keystone user-role-add --user neutron --role admin --tenant services
```

The basic network implementation in OpenStack is made of a self-service virtual data center infrastructure permitting regular users to manage one or more virtual networks within a project. Connectivity to the external networks such as Internet is provided via the physical network infrastructure. Following concepts are introduced:

  * **Tenant networks**: networks providing connectivity to instances whithin a project. Regular users can manage project networks with the allocation that an administrator defines for for them. Tenant networks can use VLAN, GRE, or VXLAN transport methods depending on the allocation. Tenant networks generally use private IP address ranges and lack connectivity to external networks. IP addresses on the project networks are private IP space within the project and for this reason, they can overlap between different projects. An embedded DHCP service assignes the IP addresses to the Virtual Machines within the project.

  * **External network**: networks providing connectivity to external networks such as the Internet. Only administrative users can manage external networks because they use the physical network infrastructure. External networks can use Flat or VLAN transport methods depending on the physical network infrastructure and generally use public IP address ranges. A flat network essentially uses the untagged frames. Similar to physical networks, only one flat network can exist. In most cases, production deployments should use VLAN transport for external networks instead of a single flat network.

  * **Routers**: typically connect tenant and external networks by implementing source NAT to provide outbound external connectivity for instances on tenant networks. Each router uses an IP address in the external network allocation for source NAT. Routers also use destination NAT to provide inbound external connectivity for instances on tenant networks. The IP addresses on routers that provide inbound external connectivity for instances on tenant networks are refered as floating IP addresses. Routers can also connect tenant networks that belong to the same project.

  * **Provider networks**: networks providing connectivity to instances by mapping directly to an existing physical network in the data center. Provider networks generally offer simplicity, performance, and reliability at the cost of flexibility. Unlike tenant networks, only administrators can manage provider networks because they require configuration of physical network infrastructure. Also, provider networks lack the concept of fixed and floating IP addresses because they only handle layer-2 connectivity for the instances running on Compute nodes. Network types for provider networks are flat (untagged) and VLAN (tagged). It is possible to allow provider networks to be shared among tenants as part of the network creation process.

###Setup Neutron networking
Install and configure Neutron services as follow

|Service|Configuration File(s)|Host Role
|-------|---------------------|---------------------|
|Neutron Server|neutron.conf, ml2_conf.ini|Controller Node|
|Neutron Open vSwitch Agent|openvswitch_agent.ini|Network Node, Compute Node|
|Neutron L3 Agent|l3_agent.ini|Network Node|
|Neutron DHCP Agent|dhcp_agent.ini|Network Node|
|Neutron Metadata Agent|metadata_agent.ini|Network Node|

On the Controller node, install the Neutron packages
```
# yum -y install openstack-neutron
# yum -y install openstack-neutron-ml2
```

Configure the Neutron service by editing the ``/etc/neutron/neutron.conf`` configuration file
```
# vi /etc/neutron/neutron.conf
[DEFAULT]
...
verbose = True
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
dhcp_agent_notification = True
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
auth_strategy = keystone
rpc_backend = rabbit
...
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = <password>
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = <service password>

[database]
connection = mysql://neutron:<password>@controller/neutron_ml2

[nova]
auth_url = http://controller:35357
auth_plugin = <password>
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = nova
password = <service password>

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = <rabbit password>
```

Configure the ML2 plugin by editing the ``/etc/neutron/plugin.ini`` initialization file. We are going to configure the **VxLAN** for tunnel encapsulation of tenant networks. Other options are: **VLAN** and **GRE**.
```
[ml2]
type_drivers = flat, vxlan, vlan, gre
# we are going to configure the vxlan for tunnel encapsulation of tenant networks;
# other options are: vlan and gre;
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = external
# use flat_networks = * to allow flat networks with arbitrary names

[ml2_type_vxlan]
vni_ranges = 1001:2000
vxlan_group =239.1.1.2

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_ipset = True
```

Start and enable the Neutron service
```
# systemctl start neutron-server 
# systemctl enable neutron-server 
```

On the Controller node, update Nova compute service to use the Neutron service by adding the following lines to the ``/etc/neutron/nova.conf`` configuration file

```
# vi /etc/nova/nova.conf
[DEFAULT]
...
network_api_class=nova.network.neutronv2.api.API
security_group_api=neutron

[neutron]
url=http://controller:9696
auth_strategy=keystone
admin_auth_url=http://controller:35357/v2.0
admin_tenant_name=service
admin_username=neutron
admin_password= <service password>
```

and restart the Nova service
```
# systemctl restart openstack-nova-api
# systemctl restart openstack-nova-compute
```

On the Network node, install the Neutron packages
```
# yum install -y openstack-neutron
# yum install -y openstack-neutron-ml2
# yum install -y openstack-neutron-openvswitch
```

And configure the Neutron service by editing the ``/etc/neutron/neutron.conf`` configuration file
```
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
allow_overlapping_ips = True
rpc_backend = rabbit

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = <password>
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = <service password>

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = <password>
```

Configure the Neutron Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
local_ip = LOCAL_TUNNEL_INTERFACE_IP_ADDRESS
bridge_mappings = external:br-ex

[agent]
tunnel_types = vxlan
l2_population = True
prevent_arp_spoofing = True

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

Configure the Neutron L3 Agent by editing the ``/etc/neutron/l3_agent.ini`` initialization file
```
[DEFAULT]
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
```

Configure the Neutron DHCP Agent by editing the ``/etc/neutron/dhcp_agent.ini`` initialization file
```
[DEFAULT]
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```

Configure the Neutron Metadata Agent by editing the ``/etc/neutron/metadata_agent.ini`` initialization file
```
[DEFAULT]
verbose = True
nova_metadata_ip = controller
#Nova service on Compute nodes must use the same shared secret
metadata_proxy_shared_secret = <metadata shared secret>
```

Finally, start and enable the Neutron agents
```
# systemctl start neutron-openvswitch-agent
# systemctl start neutron-l3-agent
# systemctl start neutron-dhcp-agent
# systemctl start neutron-metadata-agent

# systemctl enable neutron-openvswitch-agent
# systemctl enable neutron-l3-agent
# systemctl enable neutron-dhcp-agent
# systemctl enable neutron-metadata-agent
```

On all the Compute nodes, install the Neutron package
```
# yum install -y openstack-neutron-openvswitch
```

and configure Neutron by editing the ``/etc/neutron/neutron.conf`` configuration file
```
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
allow_overlapping_ips = True
rpc_backend = rabbit

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = <password>
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = <service password>

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = <password>
```

Configure the Open vSwitch Agent by editing the ``/etc/neutron/plugins/ml2/openvswitch_agent.ini`` initialization file
```
[ovs]
local_ip = local_ip = LOCAL_TUNNEL_INTERFACE_IP_ADDRESS

[agent]
tunnel_types = vxlan
l2_population = True
prevent_arp_spoofing = True

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```
Finally, start and enable the Neutron agents
```
# systemctl start neutron-openvswitch-agent
# systemctl enable neutron-openvswitch-agent
```

On all the Compute nodes, update Nova compute service to use the Neutron service by adding the following lines to the ``/etc/neutron/nova.conf`` configuration file

```
[DEFAULT]
...
network_api_class=nova.network.neutronv2.api.API
security_group_api=neutron
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver=nova.virt.firewall.NoopFirewallDriver
metadata_listen=0.0.0.0
metadata_host=controller
vif_plugging_is_fatal=True
vif_plugging_timeout=300

[neutron]
service_metadata_proxy=True
#Metadata Agent on Network node must use the same shared secret
metadata_proxy_shared_secret = <metadata shared secret>
url=http://controller:9696
auth_strategy=keystone
admin_auth_url=http://controller:35357/v2.0
admin_tenant_name=service
admin_username=neutron
admin_password= <service password>
default_tenant_id=default
```

and restart the Nova service
```
# systemctl restart openstack-nova-api
# systemctl restart openstack-nova-compute
```

Check the list of Agents
```
# neutron agent-list
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
| 14dbe6a2-7e8a-48ff-80b6-6dac6c6220a0 | DHCP agent         | network   | :-)   | True           | neutron-dhcp-agent        |
| 21fda60f-739e-4c2c-8235-a90847d6c347 | Open vSwitch agent | compute01 | :-)   | True           | neutron-openvswitch-agent |
| 55cc04e9-b0a9-4c26-a712-6dcab8ef2351 | Open vSwitch agent | compute02 | :-)   | True           | neutron-openvswitch-agent |
| 75968236-ff9c-44cb-a982-f93bcded63f4 | Loadbalancer agent | network   | :-)   | True           | neutron-lbaas-agent       |
| 98e68a68-1a73-47e7-938c-d3e0d0f36173 | Metadata agent     | network   | :-)   | True           | neutron-metadata-agent    |
| 9fb1d4f9-4a34-4d70-8823-a5ed17124618 | Open vSwitch agent | network   | :-)   | True           | neutron-openvswitch-agent |
+--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```

###Configure the external network
In the basic networking scenario with a single flat external network, only administrative users can manage external networks because they use the physical network infrastructure. In this example, we are going to create a shared external network to be used by all tenants.

On the Control node, login as ``admin`` Keystone user and create the external network
```
# source keystonerc_admin
# neutron net-create external-flat-network \
--shared \
--provider:network_type flat \
--provider:physical_network external \
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
| provider:physical_network | external                             |
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
* ``provider:physical_network external``
* ``router:external True``
* ``shared``

The external network shares the same subnet and gateway associated with the physical network connected to the external interface on the network node. Specify an exclusive slice of this subnet for router and IP addresses to prevent interference with other devices on the same external network

```
# neutron subnet-create external-flat-network 172.16.1.0/24  \
--name external-flat-subnetwork \
--gateway 172.16.1.1 \
--disable-dhcp \
--allocation-pool start=172.16.1.200,end=172.16.1.220

Created a new subnet:
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "172.16.1.200", "end": "172.16.1.220"} |
| cidr              | 172.16.1.0/24                                    |
| dns_nameservers   |                                                  |
| enable_dhcp       | False                                            |
| gateway_ip        | 172.16.1.1                                       |
| host_routes       |                                                  |
| id                | 40f89cb3-9474-48e0-ab4c-7fa3fb57009e             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | external-flat-subnetwork                         |
| network_id        | 6ede0952-25f7-489d-9ce4-0126da7cb7d0             |
| subnetpool_id     |                                                  |
| tenant_id         | 5ccf7027366442709bde78831da6cce2                 |
+-------------------+--------------------------------------------------+
```

###Configure Tenant networks
The tenant networks are created by the tenant users. In this section, we are going to configure a Tenant network based on VxLAN as tunneling protocol. Login as a tenant user and create a tenant network
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

###Configure Security Groups
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

###Configure GRE Tunnel encapsulation for Tenant networks
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
# set the local IP as the IP address of the NIC where GRE tunnel will be initiated
local_ip = 192.168.1.38
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
# set the local IP as the IP address of the NIC where GRE tunnel will be initiated
local_ip = 192.168.1.32
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

###Configure VLANs for Tenant networks
In this section we are going to use a VLAN L2 switch to implement the Tenant networks. The switch must support the VLAN trunking in order to get working.




