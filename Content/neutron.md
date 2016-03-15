###Neutron Networking Service
OpenStack administrators can configure rich network topologies by creating and configuring networks and subnets. In particular, OpenStack networking supports each tenant having multiple private networks, and allows tenants to choose their own IP addressing space, even if those IP addresses overlap with those used by other tenants. This enables advanced cloud networking use cases, such as building multitiered web applications and allowing applications to be migrated to the cloud without changing IP addresses.

OpenStack networking uses the concept of a plug-in, which is a pluggable back-end implementation of the OpenStack networking API. A plug-in can use a variety of technologies. Some OpenStack networking plug-ins might use basic Linux networking, while others might use more advanced technologies, such as Open VSwitch, to provide similar benefits.

####Implementing Neutron Service with ML2 plugin and Open VSwitch
On the Controller node, create the Neutron service
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

# keystone endpoint-create --service-id fe006cc01d234e93a308933d60c396f2 --publicurl http://10.10.10.30:9696 --adminurl http://10.10.10.30:9696 --internalurl http://10.10.10.30:9696
# keystone user-create --name neutron --pass <password>
# keystone user-role-add --user neutron --role admin --tenant services
```

On the Controller node, install the Neutron server and plug-in components. This example chooses ML2 plugin with Open vSwitch 
```
# yum -y install openstack-neutron
# yum -y install openstack-neutron-ml2
```

On the Controller node, configure the Neutron service by editing the ``/etc/neutron/neutron.conf`` configuration file
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

####Basic Neutron scenario with a flat external network
The basic network implementation in OpenStack is made of a self-service virtual data center infrastructure permitting regular users to manage one or more virtual networks within a project (aka **Tenant** networks). Connectivity to the external networks such as Internet is provided via the physical network infrastructure (aka **External** networks).

**Tenant networks**: networks providing connectivity to instances whithin a project. Regular users can manage project networks with the allocation that an administrator defines for for them. Tenant networks can use VLAN, GRE, or VXLAN transport methods depending on the allocation. Tenant networks generally use private IP address ranges and lack connectivity to external networks. IP addresses on the project networks are private IP space within the project and for this reason, they can overlap between different projects. An embedded DHCP service assignes the IP addresses to the Virtual Machines within the project.

**External network**: networks providing connectivity to external networks such as the Internet. Only administrative users can manage external networks because they use the physical network infrastructure. External networks can use Flat or VLAN transport methods depending on the physical network infrastructure and generally use public IP address ranges. A flat network essentially uses the untagged frames. Similar to physical networks, only one flat network can exist. In most cases, production deployments should use VLAN transport for external networks instead of a single flat network.

**Routers**: typically connect project and external networks by implementing source NAT to provide outbound external connectivity for instances on project networks. Each router uses an IP address in the external network allocation for source NAT. Routers also use destination NAT to provide inbound external connectivity for instances on project networks. The IP addresses on routers that provide inbound external connectivity for instances on project networks are refered as floating IP addresses. Routers can also connect project networks that belong to the same project.

In this section, we are going to creates one flat external network and project (tenant) networks using VxLAN.



