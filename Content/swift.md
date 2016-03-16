###Swift Object Storage

The object storage service provides object storage in virtual containers, allowing users to store and retrieve objects such as files without a filesystem interface. The service's distributed architecture supports horizontal scaling; object redundancy is provided through software-based data replication. By supporting eventual consistency through asynchronous replication, it is well-suited to multiple datacenter deployment.
Object storage uses the concept of:

* **Storage replicas** Used to maintain the state of objects in the case of outage. A minimum of three replicas is recommended.

* **Storage zones** Used to host replicas. Zones ensure that each replica of a given object can be stored separately. A zone might represent an individual disk drive or array, a server, all the servers in a rack, or even an entire datacenter.

* **Storage regions** Essentially a group of zones sharing a location. Regions can be groups of servers or server farms, usually located in the same geographical area. Regions have a separate API endpoint per object storage service installation, which allows for discrete separation of services.

The OpenStack object storage service is a modular service with the following components:

* _openstack-swift-proxy_ The proxy service uses the object ring to decide where to store newly uploaded objects. It updates the relevant container database to reflect the presence of a new object. If a newly uploaded object goes to a new container, the proxy service also updates the relevant account database to reflect the new container. The proxy service also directs get requests to one of the nodes where a replica of the requested object is stored, either randomly or based on response time from the node. It provides access to the public Swift API, and is responsible for handling and routing requests. The proxy server streams objects to users without spooling. Objects can also be served via HTTP.

* _openstack-swift-object_ The object service is responsible for storing data objects in partitions on disk devices. Each partition is a directory, and each object is held in a subdirectory of its partition directory. A MD5 hash of the path to the object is used to identify the object itself. The service stores, retrieves, and deletes objects.

* _openstack-swift-container_ The container service maintains databases of objects in containers. There is one database file for each container, and they are replicated across the cluster. Containers are defined when objects are put in them. Containers make finding objects faster by limiting object listings to specific container namespaces. The container service is responsible for listings of containers using the account database.

* _openstack-swift-account_ The account service maintains databases of all of the containers accessible by any given account. There is one database file for each account, and they are replicated across the cluster. Any account has access to a particular group of containers. An account maps to a tenant in the identity service. The account service handles listings of objects (what objects are in a specific container) using the container database.

All of the services can be installed on each node or, alternatively, on dedicated machines. In addition, the following components are in place for proper operation:

* **Ring files** Contain details of all the storage devices, and are used to deduce where a particular piece of data is stored (maps the names of stored entities to their physical location). One file is created for each object, account, and container server.

* **Object storage** With either the ext4 or the XFS (recommended by Red Hat) file system. The mount point is expected to be /srv/node.

* **Housekeeping processes** For example, replication and auditors.

###Implementing the Swift Object Storage Service

On the Controller node, install the necessary components for the Swift object storage service, including the swift client and memcached

```
# yum install -y openstack-swift-proxy
# yum install -y python-swiftclient
# yum install -y memcached
```

Source the Keystone environment variables with the authentication information and create a swift user with the admin role. Then create a services tenant and add the swift user to it.

```
# source /root/keystonerc_admin
[~(keystone_admin)]$ keystone user-create --name swift --pass <password>
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | ce28fda90edc4e7bbb8c0abbe5e76b06 |
|   name   |              swift               |
| username |              swift               |
+----------+----------------------------------+
[~(keystone_admin)]$ keystone tenant-create --name services
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |                                  |
|   enabled   |               True               |
|      id     | fb18c3a9c2134a999f40bd2310d9da0f |
|     name    |             services             |
+-------------+----------------------------------+
[~(keystone_admin)]$ keystone user-role-add --role admin --tenant services --user swift
```

Create the Swift Object Storage Service
```
[~(keystone_admin)]# keystone service-create --name swift --type object-store --description "Swift Object Storage Service"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |   Swift Object Storage Service   |
|   enabled   |               True               |
|      id     | 350e61c1866d4f468fa8ded69ed848d1 |
|     name    |              swift               |
|     type    |           object-store           |
+-------------+----------------------------------+
```

Create the end point for the service you just declared (make sure to use the ID the previous command returned):
```
[~(keystone_admin)]$ keystone endpoint-create --service-id 350e61c1866d4f468fa8ded69ed848d1 --publicurl "http://controller:8080/v1/AUTH_%(tenant_id)s" --adminurl "http://controller:8080/v1/AUTH_%(tenant_id)s" --internalurl "http://controller:8080/v1/AUTH_%(tenant_id)s"
+-------------+---------------------------------------------+
|   Property  |                    Value                    |
+-------------+---------------------------------------------+
|   adminurl  | http://controller:8080/v1/AUTH_%(tenant_id)s|
|      id     |       142b83a38cf44969b9ad81759db8384c      |
| internalurl | http://controller:8080/v1/AUTH_%(tenant_id)s|
|  publicurl  | http://controller:8080/v1/AUTH_%(tenant_id)s|
|    region   |                  regionOne                  |
|  service_id |       350e61c1866d4f468fa8ded69ed848d1      |
+-------------+---------------------------------------------+
```

Check the services in Keystone

```
[~(keystone_admin)]$ keystone service-list
+----------------------------------+----------+--------------+------------------------------+
|                id                |   name   |     type     |         description          |
+----------------------------------+----------+--------------+------------------------------+
| 623343aa351e4c5fb726e0932a89bbfb | keystone |   identity   |  Keystone Identity Service   |
| 350e61c1866d4f468fa8ded69ed848d1 |  swift   | object-store | Swift Object Storage Service |
+----------------------------------+----------+--------------+------------------------------+
```

###Deploying the Swift Object Storage Service
The object storage service stores objects on the file system, usually on a number of connected physical storage devices. All of the devices that will be used for object storage must be formatted with either ext4 or XFS, and mounted under the ``/srv/node/`` directory. 

Each Storage node in the Swift cluster needs to have the following packages installed:
```
# yum install -y openstack-swift-object
# yum install -y openstack-swift-container
# yum install -y openstack-swift-account
```

Configuring a Swift cluster made of multiple storage nodes, copy the ``/etc/swift`` directory to all nodes of the cluster from the first node. In this section, we are going to configure the Storage node as Swift cluster. To keep things simple, our cluster will be made of only one node containing three separate zones for data redundancy (3 replicas). In a production envinronment, the Swift cluster should be made of three separate nodes. The Storage node has 3 logical volumes, already be formatted with the XFS file system.

Create the mount points and mount the volumes persistently to the appropriate directories and set the ownership to the swift user.

```
# vgs
  VG             #PV #LV #SN Attr   VSize  VFree
  os               1   3   0 wz--n- 73.42g    0
  swift01          1   1   0 wz--n- 48.83g    0
  swift02          1   1   0 wz--n- 48.83g    0
# pvs
  PV         VG             Fmt  Attr PSize  PFree
  /dev/sda2  os             lvm2 a--  73.42g    0
  /dev/sda5  swift01        lvm2 a--  48.83g    0
  /dev/sda6  swift02        lvm2 a--  48.83g    0
# lvs
  LV     VG             Attr       LSize  Pool Origin Data%  Move Log Cpy%Sync Convert
  data   os             -wi-ao---- 19.53g
  root   os             -wi-ao---- 50.00g
  swap   os             -wi-ao----  3.89g
  zone01 swift01        -wi-ao---- 48.83g
  zone02 swift02        -wi-ao---- 48.83g

# mkdir -p /srv/node/z1d1
# mkdir -p /srv/node/z2d1
# cp /etc/fstab /etc/fstab.orig
# vi /etc/fstab

/dev/mapper/os-root         /                       xfs     defaults        1 1
UUID=xyz                    /boot                   xfs     defaults        1 2
/dev/mapper/os-data         /data                   xfs     defaults        1 2
/dev/mapper/os-swap         swap                    swap    defaults        0 0
/dev/mapper/swift01-zone01  /srv/node/z1d1          xfs     defaults        0 0
/dev/mapper/swift02-zone02  /srv/node/z2d1          xfs     defaults        0 0

# mount -a
# chown -R swift:swift /srv/node/
```
Restore the SELinux context of /srv

``# restorecon -vR /srv``

Configure the Swift Object Storage Service Ring. Three ring files need to be created: one to track the objects stored by the object storage service, one to track the containers that objects are placed in, and one to track which accounts can access which containers. The ring files are used to deduce where a particular piece of data is stored.

```
# source /root/keystonerc_admin
[~(keystone_admin)]# swift-ring-builder /etc/swift/account.builder create 12 2 1
[~(keystone_admin)]# swift-ring-builder /etc/swift/container.builder create 12 2 1
[~(keystone_admin)]# swift-ring-builder /etc/swift/object.builder create 12 2 1
```
Add the devices to the container ring

```
[~(keystone_admin)]# swift-ring-builder /etc/swift/account.builder add z1-10.10.10.97:6002/z1d1 100
[~(keystone_admin)]# swift-ring-builder /etc/swift/account.builder add z2-10.10.10.97:6002/z2d1 100
[~(keystone_admin)]# swift-ring-builder /etc/swift/container.builder add z1-10.10.10.97:6001/z1d1 100
[~(keystone_admin)]# swift-ring-builder /etc/swift/container.builder add z2-10.10.10.97:6001/z2d1 100
[~(keystone_admin)]# swift-ring-builder /etc/swift/object.builder add z1-10.10.10.97:6000/z1d1 100
[~(keystone_admin)]# swift-ring-builder /etc/swift/object.builder add z2-10.10.10.97:6000/z2d1 100
```

Distribute the partitions across the drives in the ring

```
[~(keystone_admin)]# swift-ring-builder /etc/swift/account.builder rebalance
[~(keystone_admin)]# swift-ring-builder /etc/swift/container.builder rebalance
[~(keystone_admin)]# swift-ring-builder /etc/swift/object.builder rebalance
[~(keystone_admin)]#
[~(keystone_admin)]# ls -lrt /etc/swift/*gz
-rw-r--r--. 1 root root 1851 Jan 22 15:47 /etc/swift/account.ring.gz
-rw-r--r--. 1 root root 1841 Jan 22 15:47 /etc/swift/container.ring.gz
-rw-r--r--. 1 root root 1834 Jan 22 15:47 /etc/swift/object.ring.gz
```

Start up the services and make them persistent at boot
```
[~(keystone_admin)]# systemctl start openstack-swift-account
[~(keystone_admin)]# systemctl enable openstack-swift-account
[~(keystone_admin)]# systemctl start openstack-swift-container
[~(keystone_admin)]# systemctl enable openstack-swift-container
[~(keystone_admin)]# systemctl start openstack-swift-object
[~(keystone_admin)]# systemctl enable openstack-swift-object
```

All files in the /etc/swift directory should be owned by root:swift

``[~(keystone_admin)]# chown -R root:swift /etc/swift``

Check for any error

``# grep ERROR /var/log/messages``


Enable the memcached and openstack-swift-proxy services permanently.

```
# systemctl start memcached
# systemctl enable memcached
# systemctl start openstack-swift-proxy
# systemctl enable openstack-swift-proxy
```
And check the services
```
# openstack-status
== Keystone service ==
openstack-keystone:                     active
== Swift services ==
openstack-swift-proxy:                  active
openstack-swift-account:                active
openstack-swift-container:              active
openstack-swift-object:                 active
== Support services ==
dbus:                                   active
rabbitmq-server:                        active
memcached:                              active
== Keystone users ==
Warning keystonerc not sourced
#
#  netstat -plnt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      2471/master
tcp        0      0 0.0.0.0:35357           0.0.0.0:*               LISTEN      1393/python
tcp        0      0 0.0.0.0:5000            0.0.0.0:*               LISTEN      1393/python
tcp        0      0 0.0.0.0:25672           0.0.0.0:*               LISTEN      1392/beam.smp
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      3138/mysqld
tcp        0      0 0.0.0.0:11211           0.0.0.0:*               LISTEN      1395/memcached
tcp        0      0 172.25.1.10:6000        0.0.0.0:*               LISTEN      7000/python
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      6859/python
tcp        0      0 172.25.1.10:6001        0.0.0.0:*               LISTEN      6979/python
tcp        0      0 0.0.0.0:4369            0.0.0.0:*               LISTEN      1455/epmd
tcp        0      0 172.25.1.10:6002        0.0.0.0:*               LISTEN      6956/python
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1402/sshd
tcp6       0      0 ::1:25                  :::*                    LISTEN      2471/master
tcp6       0      0 :::5672                 :::*                    LISTEN      1392/beam.smp
tcp6       0      0 :::11211                :::*                    LISTEN      1395/memcached
tcp6       0      0 :::22                   :::*                    LISTEN      1402/sshd

```
Check for any error

``# grep ERROR /var/log/messages``

###Working with the Swift Object Storage Service
After starting succesfully the service, it is possible to store and retrive data in containers using the CLI Swift client.

Source the credentials and query the service using the CLI client

```
# source /root/keystonerc_admin
[~(keystone_admin)]$ swift stat
        Account: AUTH_a6c6d1f171e649ba8af7a7a6cf63c3b9
     Containers: 0
        Objects: 0
          Bytes: 0
X-Put-Timestamp: 1421939769.00333
    X-Timestamp: 1421939769.00333
     X-Trans-Id: tx0df39bb7df404386aa13b-0054c11438
   Content-Type: text/plain; charset=utf-8

```

Create two containers and upload some content

```
[~(keystone_admin)]$ swift post container01
[~(keystone_admin)]$ swift post container02
[~(keystone_admin)]$
[~(keystone_admin)]$ swift upload container01 somedata.file
[~(keystone_admin)]$ swift upload container01 somedata2.file
[~(keystone_admin)]$ swift upload container02 somedata3.file
```

View the list and content of containers
```
[~(keystone_admin)]$ swift list
container01
container02
[~(keystone_admin)]$ swift list container01
somedata.file
somedata2.file
[~(keystone_admin)]$ swift list container02
somedata3.file
[~(keystone_admin)]$ swift stat
                        Account: AUTH_a6c6d1f171e649ba8af7a7a6cf63c3b9
                     Containers: 2
                        Objects: 1
                          Bytes: 1024
Containers in policy "policy-0": 2
   Objects in policy "policy-0": 1
     Bytes in policy "policy-0": 1024
    X-Account-Project-Domain-Id: default
                    X-Timestamp: 1421940332.49074
                     X-Trans-Id: tx3c6abaa53cfb44a485b71-0054c11793
                   Content-Type: text/plain; charset=utf-8
                  Accept-Ranges: bytes
[root@caldera01 ~(keystone_admin)]$
```

Delete the containers and their content

```
[~(keystone_admin)]$ swift delete container01
somedata.file
somedata2.file
[~(keystone_admin)]$ swift delete container02
somedata3.file
[~(keystone_admin)]$ swift list
[~(keystone_admin)]$
```
