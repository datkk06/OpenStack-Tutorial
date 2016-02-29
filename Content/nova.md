###Nova Computing Service
The Controller node is the node that runs most of the Nova services, especially the nova-scheduler, which coordinates the activities of the various Nova services. The Compute node runs the virtualization software to launch and manage instances for OpenStack.

Install and configure the Nova components on the Controller node
```
# yum install -y openstack-nova
```

The configuration files ``/etc/nova/nova.conf`` and ``/etc/nova/api-paste.ini`` contain all the relevant parameters for the Nova service.
```
# cp /etc/nova/nova.conf /etc/nova/nova.conf.orig
# cp /etc/nova/api-paste.ini /etc/nova/api-paste.ini.orig
```

Source the admin user and setup the Nova service
```
# openstack-db --init --service nova --password <password> --rootpw <password>
```
