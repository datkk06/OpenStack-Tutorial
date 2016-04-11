###Nova Computing Service
The OpenStack Compute service allows to control an Infrastructure-as-a-Service (IaaS) Cloud Computing platform. It provides control over instances and networks, and allows to manage the cloud through users and projects. The Nova Compute Service in OpenStack does not include virtualization software. Instead, it defines drivers that interact with underlying virtualization mechanisms running on host operating systems, and exposes functionality over a web-based API.

The Compute Service provides:

1. **openstack-nova-api**. Provides the OpenStack Compute API service. The Controller node hosts an instance of the service and shoud be pointed to by the Identity service endpoint definition for the Compute service.

2. **openstack-nova-compute**. Provides the OpenStack Compute service.

3. **openstack-nova-scheduler**. Provides the Compute scheduler service. The scheduler handles scheduling of requests made to the API across all the available Compute nodes.

4. **openstack-nova-conductor**. Handles database requests made by Compute nodes, ensuring that individual Compute node do not require direct database access. 


The Controller node is the node that runs most of the Nova services, especially the nova-scheduler, which coordinates the activities of the various Nova services. The Compute node runs the virtualization software to launch and manage instances for OpenStack.

Install and configure the Nova components on the Controller node
```
# yum install -y openstack-nova
```

Source the admin user and setup the Nova service
```
# source /root/keystonerc_admin
# openstack-db --init --service nova --password <password> --rootpw <password>
```

Create the nova user and the services. 
```
# keystone user-create --name compute --pass <password>
# keystone user-role-add --user compute --role admin --tenant services
# keystone service-create --name compute --type compute --description "Openstack Compute Service"
# keystone service-list
+----------------------------------+------------+---------------+---------------------------------+
|                id                |    name    |      type     |           description           |
+----------------------------------+------------+---------------+---------------------------------+
| 12d24a91d5a54add9fd684ed94f205dd |  keystone  |    identity   |    OpenStack Identity Service   |
| 57cc90c8c9024957bcebe13b26a65149 |    nova    |    compute    |    Openstack Compute Service    |
+----------------------------------+------------+---------------+---------------------------------+

# keystone endpoint-create \
   --service compute
   --publicurl "http://controller:8774/v2/%(tenant_id)s" \
   --adminurl "http://controller:8774/v2/%(tenant_id)s" \
   --internalurl "http://controller:8774/v2/%(tenant_id)s"
   --region 'RegionOne'
```

On the Controller node, the configuration file ``/etc/nova/nova.conf`` contains all the relevant parameters for the Nova service. Edit the configuration file
```
# vi /etc/nova/nova.conf

[DEFAULT]
state_path=/var/lib/nova
enabled_apis=osapi_compute,metadata
osapi_compute_listen=0.0.0.0
osapi_compute_listen_port=8774
rootwrap_config=/etc/nova/rootwrap.conf
api_paste_config=api-paste.ini
auth_strategy=keystone
log_dir=/var/log/nova
scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
notification_driver=nova.openstack.common.notifier.rpc_notifier
rpc_backend=rabbit

network_api_class=nova.network.neutronv2.api.API
security_group_api=neutron
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver=nova.virt.firewall.NoopFirewallDriver

[glance]
host=controller
port=9292
protocol=http

[oslo_concurrency]
lock_path=/var/lib/nova/tmp

[oslo_messaging_rabbit]
rabbit_host=controller
rabbit_port=5672
rabbit_userid=guest
rabbit_password=guest

[database]
connection=mysql://nova:<password>@localhost/nova

[keystone_authtoken]
auth_uri=http://controller:5000
auth_url=http://controller:35357
auth_plugin=password
project_domain_id=default
user_domain_id=default
project_name=service
username=nova
password=<keystone service password>

[neutron]
url=http://controller:9696
auth_strategy=keystone
admin_auth_url=http://controller:35357/v2.0
admin_tenant_name=service
admin_username=neutron
admin_password=<neutron service password>
```

