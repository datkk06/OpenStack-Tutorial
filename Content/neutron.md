###Neutron Networking Service
OpenStack administrators can configure rich network topologies by creating and configuring networks and subnets. In particular, OpenStack networking supports each tenant having multiple private networks, and allows tenants to choose their own IP addressing space, even if those IP addresses overlap with those used by other tenants. This enables advanced cloud networking use cases, such as building multitiered web applications and allowing applications to be migrated to the cloud without changing IP addresses.

OpenStack networking uses the concept of a plug-in, which is a pluggable back-end implementation of the OpenStack networking API. A plug-in can use a variety of technologies. Some OpenStack networking plug-ins might use basic Linux networking, while others might use more advanced technologies, such as Open VSwitch, to provide similar benefits.

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

# keystone endpoint-create --service-id fe006cc01d234e93a308933d60c396f2 --publicurl http://controller:9696 --adminurl http://10.10.10.30:9696 --internalurl http://controller:9696
# keystone user-create --name neutron --pass <password>
# keystone user-role-add --user neutron --role admin --tenant services
```

###Setup Neutron networking with a flat external network
The basic network implementation in OpenStack is made of a self-service virtual data center infrastructure permitting regular users to manage one or more virtual networks within a project. Connectivity to the external networks such as Internet is provided via the physical network infrastructure. Following concepts are introduced:

* **Tenant networks**: networks providing connectivity to instances whithin a project. Regular users can manage project networks with the allocation that an administrator defines for for them. Tenant networks can use VLAN, GRE, or VXLAN transport methods depending on the allocation. Tenant networks generally use private IP address ranges and lack connectivity to external networks. IP addresses on the project networks are private IP space within the project and for this reason, they can overlap between different projects. An embedded DHCP service assignes the IP addresses to the Virtual Machines within the project.

* **External network**: networks providing connectivity to external networks such as the Internet. Only administrative users can manage external networks because they use the physical network infrastructure. External networks can use Flat or VLAN transport methods depending on the physical network infrastructure and generally use public IP address ranges. A flat network essentially uses the untagged frames. Similar to physical networks, only one flat network can exist. In most cases, production deployments should use VLAN transport for external networks instead of a single flat network.

* **Routers**: typically connect project and external networks by implementing source NAT to provide outbound external connectivity for instances on project networks. Each router uses an IP address in the external network allocation for source NAT. Routers also use destination NAT to provide inbound external connectivity for instances on project networks. The IP addresses on routers that provide inbound external connectivity for instances on project networks are refered as floating IP addresses. Routers can also connect project networks that belong to the same project.

In this section, we are going to creates one flat external network and multiple project networks using VxLAN. All traffic between different projects or to/from the Internet flows through the Network node. Also traffic between the same project flows through the Network node if the source and destination virtual machines are hosted on different Compute nodes. Only the traffic between the same project does not reach the Network node if the source and destination virtual machines are hosted on the same Compute node. We are going to use the Open vSwitch as **Software Defined Network** implementation.

Install and configure Neutron services as follow

|Host Role|Service|Configuration File(s)|
|---------|-------|---------------------|
|Controller Node|Neutron Server|neutron.conf, ml2_conf.ini, nova.conf|
|Compute Node 01|Neutron Open vSwitch Agent|neutron.conf, nova.conf, openvswitch_agent.ini|
|Compute Node 02|Neutron Open vSwitch Agent|neutron.conf, nova.conf, openvswitch_agent.ini|
|Network Node|Neutron L3 Agent|neutron.conf, l3_agent.ini|
|Network Node|Neutron DHCP Agent|neutron.conf, dhcp_agent.ini|
|Network Node|Neutron Metadata Agent|neutron.conf, metadata_agent.ini|
|Network Node|Neutron Open vSwitch Agent|neutron.conf, openvswitch_agent.ini|

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

Configure the ML2 plugin by editing the ``/etc/neutron/plugins/ml2/ml2_conf.ini`` initialization file
```
[ml2]
type_drivers = flat,vxlan
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
metadata_proxy_shared_secret = <metadata shared secret> #Nova service on Compute nodes must use the same shared secret
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

On all the Compute nodes, install the Neutron packages
```
# yum install -y openstack-neutron
# yum install -y openstack-neutron-ml2
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
metadata_proxy_shared_secret = <metadata shared secret> #Metadata Agent on Network node must use the same shared secret
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

Cehck the list of Agents
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

###Open vSwitch L2 layout
The Open vSwitch installed on the Network node and all the Compute nodes is controlled by Neutron service via the Open vSwitch Neutron Agents. To check the layout created by Neutron use the ``ovs-vsctl show`` command.

On the Network node
```
# ovs-vsctl show
d7930874-e155-42d7-978a-f78d0bcb218e
    Bridge br-ex
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
        Port br-ex
            Interface br-ex
                type: internal
        Port "ens33"
            Interface "ens33"
    Bridge br-int
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
    Bridge br-tun
        fail_mode: secure
        Port br-tun
            Interface br-tun
                type: internal
        Port "vxlan-c0a80120"
            Interface "vxlan-c0a80120"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.38", out_key=flow, remote_ip="192.168.1.32"}
        Port "vxlan-c0a80122"
            Interface "vxlan-c0a80122"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.38", out_key=flow, remote_ip="192.168.1.34"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    ovs_version: "2.4.0"
```

On the Compute 01 node
```
# ovs-vsctl show
7bb6d324-ebd3-4c44-8448-63b6da83fc93
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-c0a80122"
            Interface "vxlan-c0a80122"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.32", out_key=flow, remote_ip="192.168.1.34"}
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-c0a80126"
            Interface "vxlan-c0a80126"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.32", out_key=flow, remote_ip="192.168.1.38"}
    Bridge br-int
        fail_mode: secure
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "2.4.0"
```

On the Compute 02 node
```
# ovs-vsctl show
e0ec8cf3-8df5-4ac5-9718-4223bcd54b0c
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-c0a80120"
            Interface "vxlan-c0a80120"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.34", out_key=flow, remote_ip="192.168.1.32"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
        Port "vxlan-c0a80126"
            Interface "vxlan-c0a80126"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.34", out_key=flow, remote_ip="192.168.1.38"}
    Bridge br-int
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
    ovs_version: "2.4.0"
```
