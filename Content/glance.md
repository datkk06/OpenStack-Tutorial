###The Glance Image Service
The Glance image server requires Keystone to be in place for identity management and authorization. It uses a MySQL database to store the metadata information for the images, such as their types, location, or size. Glance supports a variety of disk formats, such as:

|Format|Description|
|------|-----------|
|raw	|An unstructured disk image format.|
|vhd	|A common disk format used by virtual machine monitors from VMware, Xen, Microsoft, VirtualBox, and others.|
|vmdk	|Another common disk format supported by many common virtual machine monitors.|
|vdi	|A disk format supported by VirtualBox virtual machine monitor and the QEMU emulator.|
|iso	|An archive format for the data contents of an optical disc (e.g., CD-ROM).|
|qcow2	|A disk format supported by the QEMU emulator that can expand dynamically and supports the Copy on Write feature.|
|aki	|This indicates what is stored in Glance is an Amazon kernel image.|
|ari	|This indicates what is stored in Glance is an Amazon ramdisk image.|
|ami	|This indicates what is stored in Glance is an Amazon machine image.|

Glance also supports a variety of container formats:

|Format|Description|
|------|-----------|
|bare	|This indicates there is no container or metadata envelope for the image.|
|ovf	|This is the OVF container format.|
|ova	|This indicates what is stored in Glance is an OVA tar archive file.|
|aki	|This indicates what is stored in Glance is an Amazon kernel image.|
|ari	|This indicates what is stored in Glance is an Amazon ramdisk image.|
|ami	|This indicates what is stored in Glance is an Amazon machine image.|

The container format string is not currently used by Glance or other OpenStack components, so it is safe to simply specify bare as the container format if you are unsure about which format to use.

###Implementing the Glance Image Service

Install and init the service

```
# yum install -y openstack-glance
# openstack-db --init --service glance --password <password>
```

Update the Glance api configuration to use Keystone as the identity service
```
vi /etc/glance/glance-api.conf

[database]
# specify controller node
connection=mysql://glance:<glance_db_password>@controller/glance

[keystone_authtoken]
auth_uri=http://controller:5000
auth_url=http://controller:35357
auth_plugin=password
project_domain_id=default
user_domain_id=default
project_name=service
username=glance
password=<glance_service_password>
flavor=keystone

```

Update the Glance registry configuration to use Keystone as the identity service
```
vi /etc/glance/glance-registry.conf

[database]
# specify controller node
connection=mysql://glance:<glance_db_password>@controller/glance

[keystone_authtoken]
auth_uri=http://controller:5000
auth_url=http://controller:35357
auth_plugin=password
project_domain_id=default
user_domain_id=default
project_name=service
username=glance
password=<glance_service_password>
flavor=keystone

```
Start and enable the services
```
# systemctl start openstack-glance-registry
# systemctl enable openstack-glance-registry
# systemctl start openstack-glance-api
# systemctl enable openstack-glance-api
```

###Deploying the Glance Image Service
Integrate Glance with Keystone. Make sure the services tenant exists before proceeding: if there is no services tenant, create one.
```
# source ~/keystonerc_admin
[~(keystone_admin)]$ keystone tenant-list | grep services
[~(keystone_admin)]$ keystone tenant-create --name services
[~(keystone_admin)]$ keystone user-create --name glance --pass <password>
[~(keystone_admin)]$ keystone user-role-add --user glance --role admin --tenant services
[~(keystone_admin)]$ keystone service-create --name glance --type image --description "Glance Image Service"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |       Glance Image Service       |
|   enabled   |               True               |
|      id     | 1b23256221e440f3bc472da2de5aef7c |
|     name    |              glance              |
|     type    |              image               |
+-------------+----------------------------------+
[~(keystone_admin)]$ keystone service-list
+----------------------------------+----------+--------------+------------------------------+
|                id                |   name   |     type     |         description          |
+----------------------------------+----------+--------------+------------------------------+
| 1b23256221e440f3bc472da2de5aef7c |  glance  |    image     |     Glance Image Service     |
| 623343aa351e4c5fb726e0932a89bbfb | keystone |   identity   |  Keystone Identity Service   |
| 350e61c1866d4f468fa8ded69ed848d1 |  swift   | object-store | Swift Object Storage Service |
+----------------------------------+----------+--------------+------------------------------+

[~(keystone_admin)]$ keystone endpoint-create --service-id 1b23256221e440f3bc472da2de5aef7c --publicurl http://controller:9292 --adminurl http://controller:9292 --internalurl http://controller:9292
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |      http://controller:9292       |
|      id     | 8370abfa431a48c79e9005e82e72d3d7 |
| internalurl |      http://controller:9292       |
|  publicurl  |      http://controller:9292       |
|    region   |            regionOne             |
|  service_id | 1b23256221e440f3bc472da2de5aef7c |
+-------------+----------------------------------+
```
Make sure to use the correct service ID as shown in the previous command.

Allow the incoming connections by setting the firewall rules:
```
[~(keystone_admin)]$ firewall-cmd --add-port=9292/tcp --permanent
[~(keystone_admin)]$ firewall-cmd --reload
```

###Manage images using Glance
The glance command can be used to manage images. Before adding a Linux image, it is necessary to prepare the image properly with ``virt-sysprep`` command before uploading it to Glance. The command will remove SSH keys, persistent MAC addresses, and user accounts. By default, the format of the virtual machine disk is autodetected. The command is provided by the `` libguestfs-tools`` utility.

Install the utility and prepare the image
```
# yum install -y libguestfs-tools
# virt-sysprep --add <imagefile>
```
If the image is already available in the desired format, get an image
```
# cd /data
# wget http://cloud.fedoraproject.org/fedora-19.x86_64.qcow2
```
If unsure what image format has been used with a given system image, check to identify it. 
```
# qemu-img info /data/fedora-19.x86_64.qcow2
image: fedora-19.x86_64.qcow2
file format: qcow2
virtual size: 2.0G (2147483648 bytes)
disk size: 228M
cluster_size: 65536
Format specific information:
    compat: 0.10
```

In this case, the system image is already prepared, so upload it to Glance.
```
# source /root/keystonerc_admin
[~(keystone_admin)]#
[~(keystone_admin)]# glance image-create --name "fedora-19" --is-public True --disk-format qcow2 --container-format bare --file /data/fedora-19.x86_64.qcow2
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 511d9d3235079f1c381d39eb5a9fda1e     |
| container_format | bare                                 |
| created_at       | 2015-02-16T17:56:18                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 8badaf16-7fa0-44ae-97ce-807d2aa50892 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | fedora-19                            |
| owner            | a6c6d1f171e649ba8af7a7a6cf63c3b9     |
| protected        | False                                |
| size             | 239534080                            |
| status           | active                               |
| updated_at       | 2015-02-16T17:56:19                  |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
[~(keystone_admin)]# glance image-list
+--------------------------------------+-----------+-------------+------------------+-----------+--------+
| ID                                   | Name      | Disk Format | Container Format | Size      | Status |
+--------------------------------------+-----------+-------------+------------------+-----------+--------+
| 8badaf16-7fa0-44ae-97ce-807d2aa50892 | fedora-19 | qcow2       | bare             | 239534080 | active |
+--------------------------------------+-----------+-------------+------------------+-----------+--------+
```
If the system image is stored at a remote location, ``--location`` can be used to provide the URL directly without first downloading the system image. The ``--copy-from`` option is similar to the ``--location`` option, but will also populate the image in the Glance cache. The ``--is-public`` option is a Boolean switch, so it can be set to either true or false. If it is set to true, every user in Keystone can use that image inside their tenants. If set to false, only the uploading user is able to use it. 

By default, images are stored in ``/var/lib/glance/images/`` of the node where glance service is running. Check out ``/etc/glance/glance.conf`` to see what format your images are being stored in.

```
default_store=STORE
# STORE would be the format the images are stored in. The default format is file.

filesystem_store_datadir=PATH
# PATH would be where the images are. If this isn't set, it will default to /var/lib/glance/images/
```
