###High Availability
OpenStack framework currently meets high availability requirements for its own infrastructure services like MySQL backend database, RabbitMQ messaging server, Neutron services, Nova services, Cinder Services, etc. However, OpenStack does not guarantee high availability for individual guest instances. This section reports some common methods of implementing highly available systems, with an emphasis on the core OpenStack services and other open source services that are closely aligned with the OpenStack framework.

In an High Availability system, preventing single points of failure can depend on whether or not a service is stateless.

* **Stateless service**: a service that provides a response after the request and then requires no further attention. To make a stateless service highly available, you need to provide redundant instances and load balance them. Examples of OpenStack services that are stateless are: nova-api, nova-conductor, glance-api, keystone-api, neutron-api and nova-scheduler.

* **Stateful service**: a service where subsequent requests to the service depend on the results of the first request. Stateful services are more difficult to manage because a single action typically involves more than one request, so simply providing redundant instances and load balancing does not solve the problem. Examples of OpenStack services that are stateful include the OpenStack MySQL database and RabbitMQ message queue server.

Two components drive High Availability for all core and non-core OpenStack services:

* **Cluster Manager**: the cluster manager is responsible for the startup and recovery of an inter-related services across a set of appropriate responses from the cluster manager to ensure service availability and data integrity. OpenStack uses **Pacemaker** as Cluster Manager. By configuring virtual IP addresses, services, and other features as resources in a cluster, Pacemaker makes sure that a set of OpenStack resources are running and available. When a service or entire node in a cluster fails, Pacemaker can restart the service, take the node out of the cluster, or reboot the node. 

* **Proxy Server**: the Proxy Server acts as load balancer between different instances of a service running on different nodes of the cluster. OpenStack uses the HAProxy as Proxy Server.

By combining the Cluster Manager and the Proxy Server capabilities, the OpenStack platform is able to provide High Availability of the infrastructure.

Highly available services in OpenStack run in one of two modes:

  1. **active/active**: in this mode, the same service is running on multiple controller nodes with Pacemaker, then traffic can either be distributed across the nodes running the requested service by HAProxy or directed to a particular controller via a single IP address. In some cases, HAProxy distributes traffic to active/active services in a round robin fashion.
  2. **active/passive**: services that are not capable of or reliable enough to run in active/active mode are run in active/passive mode. This means that only one instance of the service is active at a time.
  
