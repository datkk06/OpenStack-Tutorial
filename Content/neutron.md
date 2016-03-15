###Neutron Networking Service
OpenStack administrators can configure rich network topologies by creating and configuring networks and subnets. In particular, OpenStack networking supports each tenant having multiple private networks, and allows tenants to choose their own IP addressing space, even if those IP addresses overlap with those used by other tenants. This enables advanced cloud networking use cases, such as building multitiered web applications and allowing applications to be migrated to the cloud without changing IP addresses.

OpenStack networking uses the concept of a plug-in, which is a pluggable back-end implementation of the OpenStack networking API. A plug-in can use a variety of technologies. Some OpenStack networking plug-ins might use basic Linux VLANs and IP tables, while others might use more advanced technologies, such as L2 and L3 tunneling, to provide similar benefits.

###Implementing Neutron Service
On the Controller node, source the keystonerc_admin file to enable authentication with administrative privileges
```
# source /root/keystonerc_admin
```
Create the neutron user and service
```
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
#
# keystone catalog --service=network

Service: network
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminURL  |     http://10.10.10.30:9696      |
|      id     | 59a0ef1960db4046be8eacd2d371e682 |
| internalURL |     http://10.10.10.30:9696      |
|  publicURL  |     http://10.10.10.30:9696      |
|    region   |            RegionOne             |
+-------------+----------------------------------+

# keystone endpoint-create --service-id fe006cc01d234e93a308933d60c396f2 --publicurl http://10.10.10.30:9696 --adminurl http://10.10.10.30:9696 --internalurl http://10.10.10.30:9696
# keystone user-create --name neutron --pass <password>
# keystone user-role-add --user neutron --role admin --tenant services
```

Install the OpenStack networking components: Open vSwitch, the ML2 and Open vSwitch Neutron plug-ins:
```
# yum -y install openvswitch
# yum -y install openstack-neutron
# yum -y install openstack-neutron-ml2
# yum -y install openstack-neutron-openvswitch
```
