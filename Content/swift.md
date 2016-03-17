###Swift Object Storage

The object storage service provides object storage in virtual containers, allowing users to store and retrieve objects such as files without a filesystem interface. The service's distributed architecture supports horizontal scaling; object redundancy is provided through software-based data replication. By supporting eventual consistency through asynchronous replication, it is well-suited to multiple datacenter deployment.
Object storage uses the concept of:

* **Storage replicas** Used to maintain the state of objects in the case of outage. A minimum of three replicas is recommended.

* **Storage zones** Used to host replicas. Zones ensure that each replica of a given object can be stored separately. A zone might represent an individual disk drive or array, a server, all the servers in a rack, or even an entire datacenter.

* **Storage regions** Essentially a group of zones sharing a location. Regions can be groups of servers or server farms, usually located in the same geographical area. Regions have a separate API endpoint per object storage service installation, which allows for discrete separation of services.

The OpenStack object storage service is a modular service with the following components:

* _openstack-swift-proxy_: the proxy service uses the object ring to decide where to store newly uploaded objects. It updates the relevant container database to reflect the presence of a new object. If a newly uploaded object goes to a new container, the proxy service also updates the relevant account database to reflect the new container. The proxy service also directs get requests to one of the nodes where a replica of the requested object is stored, either randomly or based on response time from the node. It provides access to the public Swift API, and is responsible for handling and routing requests. The proxy server streams objects to users without spooling. Objects can also be served via HTTP.

* _openstack-swift-object_: the object service is responsible for storing data objects in partitions on disk devices. Each partition is a directory, and each object is held in a subdirectory of its partition directory. A MD5 hash of the path to the object is used to identify the object itself. The service stores, retrieves, and deletes objects.

* _openstack-swift-container_: the container service maintains databases of objects in containers. There is one database file for each container, and they are replicated across the cluster. Containers are defined when objects are put in them. Containers make finding objects faster by limiting object listings to specific container namespaces. The container service is responsible for listings of containers using the account database.

* _openstack-swift-account_: the account service maintains databases of all of the containers accessible by any given account. There is one database file for each account, and they are replicated across the cluster. Any account has access to a particular group of containers. An account maps to a tenant in the identity service. The account service handles listings of objects (what objects are in a specific container) using the container database.

###Implementing the Swift Object Storage Service
On the Controller node, install the necessary components for the Swift object storage service, including the swift client and memcached. In a production envinronment, the Swift proxy server shoud spawn on a couple of standalone nodes for redundancy reason. If a proxy fails, the other will take over. Howewer, in this example, we are going to install the proxy service on the Controller node. 

```
# yum install -y openstack-swift-proxy
# yum install -y python-swiftclient
# yum install -y memcached
```

Source the Keystone environment variables with the authentication information and create a swift user with the admin role. Then create a services tenant and add the swift user to it.

```
# source /root/keystonerc_admin
# openstack user create --domain default --project service --password servicepassword swift 
# openstack role add --project service --user swift admin
```

Create the Swift Object Storage Service
```
# keystone service-create --name swift object-store --description "Swift Object Storage Service"
```

Create the end point for the service you just declared
```
# openstack endpoint create --region RegionOne object-store public http://controller:8080/v1/AUTH_%\(tenant_id\)s
# openstack endpoint create --region RegionOne object-store internal http://controller:8080/v1/AUTH_%\(tenant_id\)s
# openstack endpoint create --region RegionOne object-store admin http://controller:8080/v1  
```

Check the services in Keystone
```
# keystone service-list
+----------------------------------+----------+--------------+------------------------------+
|                id                |   name   |     type     |         description          |
+----------------------------------+----------+--------------+------------------------------+
| 623343aa351e4c5fb726e0932a89bbfb | keystone |   identity   |  Keystone Identity Service   |
| 350e61c1866d4f468fa8ded69ed848d1 |  swift   | object-store | Swift Object Storage Service |
+----------------------------------+----------+--------------+------------------------------+
```

Configure the Swift service by editing the ``/etc/swift/swift.conf`` configuration file
```
# vi /etc/swift/swift.conf
[swift-hash]
swift_hash_path_suffix = swift_shared_path # it is shared among all Swift nodes, any words you like
```

Configure the Swift proxy service by editing the ``/etc/swift/proxy-server.conf`` configuration file
```
# vi /etc/swift/proxy-server.conf
...
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = swift
password = <service password>
delay_auth_decision = true
```

###Deploying the Swift Object Storage Service
The object storage service stores objects on the file system, usually on a number of connected physical storage devices. All of the devices that will be used for object storage must be formatted with either ext4 or XFS, and mounted under the ``/srv/node/`` directory. 

Configuring a Swift cluster made of multiple storage nodes, copy the ``/etc/swift`` directory to all nodes of the cluster from the first node. In this section, we are going to configure the Storage node as Swift cluster. In a production envinronment, the Swift cluster should be made of minimum 3 separate nodes, each containing a zone. To keep things simple, our cluster will be made of only one node containing 3 separate zones for data redundancy (3 replicas). Each zone is backed by a separate logical volume.

Configure the Swift Object Storage Service Ring.

The Ring maps partitions to physical locations on the disk. When any other component needs to perform any operation on an object, a container, or an account, it needs to interact with the Ring to determine the location of the object or the container in the cluster. The Ring maintains this mapping. In additions, the Ring is responsible to determine which devices are used for handoff when a failure occurs.

Three ring files need to be created. The ring files are used to deduce where a particular piece of data is stored.
1. ``object.ring`` to track the objects stored by the object storage service
2. ``container.ring`` to track the containers where the objects are placed in
3. ``account.ring`` to track which accounts (users) can access which containers.

On the Controller (proxy) node, create the Rings files

```
# source /root/keystonerc_admin
# swift-ring-builder /etc/swift/object.builder create 12 3 1
# swift-ring-builder /etc/swift/container.builder create 12 3 1
# swift-ring-builder /etc/swift/account.builder create 12 3 1
```
Add the devices to the Ring

```
# swift-ring-builder /etc/swift/object.builder add r0z0-10.10.10.30:6000/device0 100
# swift-ring-builder /etc/swift/container.builder add r0z0-10.10.10.30:6001/device0 100 
# swift-ring-builder /etc/swift/account.builder add r0z0-10.10.10.30:6002/device0 100 

# swift-ring-builder /etc/swift/object.builder add r1z1-10.10.10.30:6000/device1 100
# swift-ring-builder /etc/swift/container.builder add r1z1-10.10.10.30:6001/device1 100 
# swift-ring-builder /etc/swift/account.builder add r1z1-10.10.10.30:6002/device1 100 

# swift-ring-builder /etc/swift/object.builder add r2z2-10.10.10.30:6000/device2 100 
# swift-ring-builder /etc/swift/container.builder add r2z2-10.10.10.30:6001/device2 100 
# swift-ring-builder /etc/swift/account.builder add r2z2-10.10.10.30:6002/device2 100 
```

Distribute the partitions across the drives in the ring
```
# swift-ring-builder /etc/swift/object.builder rebalance
# swift-ring-builder /etc/swift/container.builder rebalance
# swift-ring-builder /etc/swift/account.builder rebalance

# chown swift:swift /etc/swift/*.gz 
# ls -lrt /etc/swift/*gz
-rw-r--r--. 1 swift swift 1851 Jan 22 15:47 /etc/swift/account.ring.gz
-rw-r--r--. 1 swift swift 1841 Jan 22 15:47 /etc/swift/container.ring.gz
-rw-r--r--. 1 swift swift 1834 Jan 22 15:47 /etc/swift/object.ring.gz
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



Each Storage node in the Swift cluster needs to have the following packages installed:
```
# yum install -y openstack-swift-object
# yum install -y openstack-swift-container
# yum install -y openstack-swift-account
```
On the Storage node, create the mount points and mount the logical volumes persistently to the appropriate directories and set the ownership to the swift user.

```
# lvscan | grep swift
  ACTIVE            '/dev/swift03/zone03' [16.00 GiB] inherit
  ACTIVE            '/dev/swift02/zone02' [16.00 GiB] inherit
  ACTIVE            '/dev/swift01/zone01' [16.00 GiB] inherit

# mkdir -p /srv/node/device1
# mkdir -p /srv/node/device2
# mkdir -p /srv/node/device3
# vi /etc/fstab
...
/dev/swift03/zone03     /srv/node/device3       xfs     noatime,nodiratime,nobarrier,logbufs=8  0       0
/dev/swift02/zone02     /srv/node/device2       xfs     noatime,nodiratime,nobarrier,logbufs=8  0       0
/dev/swift01/zone01     /srv/node/device1       xfs     noatime,nodiratime,nobarrier,logbufs=8  0       0

# mount -a
# chown -R swift:swift /srv/node
```


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
