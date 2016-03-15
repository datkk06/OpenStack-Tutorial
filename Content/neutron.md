###Neutron Networking Service
OpenStack administrators can configure rich network topologies by creating and configuring networks and subnets. In particular, OpenStack networking supports each tenant having multiple private networks, and allows tenants to choose their own IP addressing space, even if those IP addresses overlap with those used by other tenants. This enables advanced cloud networking use cases, such as building multitiered web applications and allowing applications to be migrated to the cloud without changing IP addresses.

OpenStack networking uses the concept of a plug-in, which is a pluggable back-end implementation of the OpenStack networking API. A plug-in can use a variety of technologies. Some OpenStack networking plug-ins might use basic Linux VLANs and IP tables, while others might use more advanced technologies, such as L2 and L3 tunneling, to provide similar benefits.

###Implementing Neutron Service
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
