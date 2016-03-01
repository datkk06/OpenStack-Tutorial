### Implementing multiple Cinder storage backends
Configuring multiple-storage backends, allows to create several backend storage solutions that serve the same OpenStack configuration and one cinder-volume service is launched for each backend storage. In a multiple-storage backend configuration, eachback end has a name. Several back ends can have the same name. In that case, the scheduler properly decides which backend the volume has to be created in.

The name of the backend is declared as an extra-specification of a volume type. When a volume is created, the scheduler chooses an appropriate backend to handle the request, according to the volume type specified by the user.
To enable a multiple-storage backends, the ``enabled_backends`` option in the cinder configuration file need to be used. This option defines the names of the configuration groups for the different backends. Each name is associated to one configuration group for a backend. The configuration group name is not related to the name of the backend.

As example, we define two LVM backends named **LOLLO** and **LOLLA** respctively. In the cinder configuration file, we are going to declare two configuration groups, one for each backend, called **lvm1** and **lvm2** respectively. We set the multiple backend option as  ``enabled_backends=lvm1,lvm2`` in the configuration file. It will look like:

```
[DEFAULT]
debug=True
verbose=True
log_dir=/var/log/cinder
use_syslog=False
auth_strategy = keystone
enabled_backends=lvm1,lvm2

rabbit_userid = guest
rabbit_password = guest
rabbit_host = caldara01
rabbit_use_ssl = false
rabbit_port = 5672

[database]
sql_connection = mysql://cinder:caldera123@controller/cinder
idle_timeout=3600
min_pool_size=1
max_retries=10
retry_interval=10

[keystone_authtoken]
admin_tenant_name = services
admin_user = cinder
admin_password = ******
auth_host = controller
auth_port = 35357
auth_protocol = http

[lvm1]
iscsi_helper=lioadm
iscsi_ip_address=192.168.2.30
volume_driver=cinder.volume.drivers.lvm.LVMISCSIDriver
volume_backend_name=LOLLA
volume_group=storage

[lvm2]
iscsi_helper=lioadm
iscsi_ip_address=192.168.2.30
volume_driver=cinder.volume.drivers.lvm.LVMISCSIDriver
volume_backend_name=LOLLO
volume_group=storage

```
To enable the new configuration, restart the cinder service
```
# openstack-service restart cinder
```
Create two new volume types and associate them with the two backends
```
# cinder type-create lvm_gold
+--------------------------------------+----------+
|                  ID                  |   Name   |
+--------------------------------------+----------+
| 60ee47a6-ebaa-4d7d-8586-b722fb00677f | lvm_gold |
+--------------------------------------+----------+
# cinder type-create lvm_silver
+--------------------------------------+------------+
|                  ID                  |    Name    |
+--------------------------------------+------------+
| e67d309c-a3e7-42a0-b8ba-f34485582734 | lvm_silver |
+--------------------------------------+------------+
# cinder type-list
+--------------------------------------+------------+
|                  ID                  |    Name    |
+--------------------------------------+------------+
| 60ee47a6-ebaa-4d7d-8586-b722fb00677f |  lvm_gold  |
| e67d309c-a3e7-42a0-b8ba-f34485582734 | lvm_silver |
+--------------------------------------+------------+
# cinder type-key lvm_silver set volume_backend_name=LOLLO
# cinder type-key lvm_gold set volume_backend_name=LOLLA

```
Now the two new backends can be used to create volumes
```
# cinder create --display-name vol1 --volume-type lvm_gold 5
# cinder create --display-name vol2 --volume-type lvm_silver 5
# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 0b1acd71-651f-41eb-a4e8-7f1fe8f39825 | available |     vol1     |  5   |   lvm_gold  |  false   |             |
| 9ae1f05e-ccf0-4532-96be-c1c21a293130 | available |     vol2     |  5   |  lvm_silver |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```

Note that each backend has its own cinder volume service
```
# cinder-manage service list
Binary           Host                                 Zone             Status     State Updated At
cinder-scheduler controller                            nova             enabled    :-)   2015-05-18 18:05:26
cinder-volume    controller@lvm2                       nova             enabled    :-)   2015-05-18 18:05:27
cinder-volume    controller@lvm1                       nova             enabled    :-)   2015-05-18 18:05:27
```

###Add a separate Storage Node
In an OpenStack production setup, one or more Storage nodes are used. This section describes how to install and configure storage nodes for the Block Storage service. The service provisions logical volumes on this device using the LVM driver and provides them to instances via iSCSI transport.

Install a Storage node and connect the Storage node with Controller node and Compute nodes using an isolate Storage network. In this example, the storage network is 192.168.2.0/24, the Management network is 10.10.10.0/24 and the Tenant network is 192.168.1.0/24.

Install the LVM package and create the volume group
```
# yum install -y lvm2
# pvcreate /dev/sdb
# vgcreate cinder-volumes /dev/sdb
Volume group "cinder-volumes" successfully created
```

Install and configure the components
```
# yum install openstack-cinder targetcli python-oslo-policy
# systemctl enable openstack-cinder-volume
# systemctl enable target
```

Edit the ``/etc/cinder/cinder.conf`` file 
```
[root@osstorage]# cat /etc/cinder/cinder.conf
[DEFAULT]
glance_host = 10.10.10.30 # controller
enable_v1_api = True
enable_v2_api = True
storage_availability_zone = nova
default_availability_zone = nova
auth_strategy = keystone
enabled_backends = lvm2
osapi_volume_listen = 0.0.0.0
osapi_volume_workers = 1
nova_catalog_info = compute:Compute Service:publicURL
nova_catalog_admin_info = compute:Compute Service:adminURL
debug = True
verbose = True
notification_driver = messagingv2
rpc_backend = rabbit
control_exchange = openstack
api_paste_config=/etc/cinder/api-paste.ini
amqp_durable_queues=False

[keystone_authtoken]
auth_uri = http://10.10.10.30:5000 #controller
auth_url = http://10.10.10.30:35357 #controller
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = cinder
password = *******

[database]
connection = mysql://cinder:*******@10.10.10.30/cinder #controller
[oslo_messaging_rabbit]
rabbit_host = 10.10.10.30 #controller
rabbit_port = 5672
rabbit_hosts = 10.10.10.30:5672 #controller
rabbit_use_ssl = False
rabbit_userid = guest
rabbit_password = guest
rabbit_ha_queues = False
heartbeat_timeout_threshold = 0
heartbeat_rate = 2
[lvm2]
iscsi_helper=lioadm
volume_group=cinder-volumes
iscsi_ip_address=192.168.2.36 #local IP on the Storage network
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
iscsi_protocol=iscsi
volume_backend_name=gold
```

Start the cinder-volume and the iscsi target services
```
# systemctl start openstack-cinder-volume
# systemctl start target
```

Check the service list
```
[root@osstorage]# cinder-manage service list
Binary           Host                                 Zone             Status     State Updated At
cinder-scheduler oscontroller                         nova             enabled    :-)   2016-03-01 14:15:34
cinder-volume    oscontroller@lvm1                    nova             enabled    :-)   2016-03-01 14:15:34
cinder-volume    osstorage@lvm2                       nova             enabled    :-)   2016-03-01 14:15:34
[root@osstorage]#
```


