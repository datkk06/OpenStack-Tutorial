### Implementing multiple storage backends
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
