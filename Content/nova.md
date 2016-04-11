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
novncproxy_host = 0.0.0.0
novncproxy_port = 6080
notify_api_faults = False
state_path=/var/lib/nova
report_interval = 10
enabled_apis = osapi_compute,metadata
osapi_compute_listen = 0.0.0.0
osapi_compute_listen_port = 8774
osapi_compute_workers = 1
metadata_listen = 0.0.0.0
metadata_listen_port = 8775
metadata_workers = 1
service_down_time = 60
rootwrap_config = /etc/nova/rootwrap.conf
volume_api_class = nova.volume.cinder.API
auth_strategy = keystone
use_forwarded_for = False
cpu_allocation_ratio = 16.0
ram_allocation_ratio = 1.5
network_api_class = nova.network.neutronv2.api.API
default_floating_pool = public
force_snat_range = 0.0.0.0/0
metadata_host = <controller>
dhcp_domain = novalocal
security_group_api = neutron
scheduler_driver = nova.scheduler.filter_scheduler.FilterScheduler
vif_plugging_is_fatal = True
vif_plugging_timeout = 300
firewall_driver = nova.virt.firewall.NoopFirewallDriver
debug = True
verbose = True
log_dir = /var/log/nova
use_syslog = False
syslog_log_facility = LOG_USER
use_stderr = True
notification_topics = notifications
rpc_backend = rabbit
amqp_durable_queues = False
sql_connection = mysql://nova:<nova db password>@<controller>/nova
image_service = nova.image.glance.GlanceImageService
lock_path = /var/lib/nova/tmp
osapi_volume_listen = 0.0.0.0
novncproxy_base_url = http://0.0.0.0:6080/vnc_auto.html

[cinder]
catalog_info = volumev2:cinderv2:publicURL

[glance]
api_servers = <controller>:9292

[keystone_authtoken]
auth_uri = http://<controller>:5000/v2.0
identity_uri = http://<controller>:35357
admin_user = nova
admin_password = <nova service password>
admin_tenant_name = services

[libvirt]
vif_driver = nova.virt.libvirt.vif.LibvirtGenericVIFDriver

[neutron]
service_metadata_proxy = True
metadata_proxy_shared_secret = <shared secret>
url = http://<controller>:9696
admin_username = neutron
admin_password = <neutron service password>
admin_tenant_name = services
region_name = RegionOne
admin_auth_url = http://<controller>:5000/v2.0
auth_strategy = keystone
ovs_bridge = br-int
extension_sync_interval = 600
timeout = 30
default_tenant_id = default

[oslo_messaging_rabbit]
rabbit_host = <controller>
rabbit_port = 5672
rabbit_hosts = <controller>:5672
rabbit_use_ssl = False
rabbit_userid = guest
rabbit_password = guest
rabbit_virtual_host = /
rabbit_ha_queues = False
heartbeat_timeout_threshold = 0
heartbeat_rate = 2

[osapi_v3]
enabled = False
```

On the Controller node, start and enable the services
```
# systemctl start openstack-nova-api
# systemctl start openstack-nova-scheduler
# systemctl start openstack-nova-cert
# systemctl start openstack-nova-conductor
# systemctl start openstack-nova-consoleauth

# systemctl enable openstack-nova-api
# systemctl enable openstack-nova-scheduler
# systemctl enable openstack-nova-cert
# systemctl enable openstack-nova-conductor
# systemctl enable openstack-nova-consoleauth
```

On each Compute node, install the Compute Service
```

```


