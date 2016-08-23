###Ceilometer Metering Service
OpenStack Ceilometer services provides metering data related to OpenStack services. It collects events and metering data by monitoring notifications from the other service services and publishes collected data to various targets including data stores and message queues.

Ceilometer main components are:

1.  The **ceilometer-compute**, an agent running on each compute node polling the local libvirt daemon to acquire performance data for the local instances, messages and emits these data via RabbitMQ.
2.  The **ceilometer-central**, an agent running on the other nodes, polling the other openstack services for statistics not related to the compute resources.
3.  The **ceilometer-notification**, a service running on the controller, responsible for consuming messages from the RabbitMQ and building events and metering data.
4.  The **ceilometer-collector**, a service running on the controller, dispatching collected telemetry data to the metering data store or other external message queues.
5.  The **ceilometer-alarm-evaluator**, running on the controller, responsible for evaluating when fire an alarm due to the associated statistic trend crossing a threshold.
6.  The **ceilometer-alarm-notifier**, running on the controller, responsible for notify alarms and trigger actions.
7.  The **ceilometer-api**, running on the controller providing access to the users.

Only the collector and the API services have access to the data store. Ceilometer service uses a NoSQL database as datastore.

####Implementing Ceilometer
On the Controller node, setup the Ceilometer service as described in OpenStack official documentation, [here](http://docs.openstack.org/liberty/install-guide-rdo/ceilometer-install.html).

To keep things simple, we are going to install only the compute agent on any of the Compute node. This agent will be able to acquire performance and metering data only from the virtual machines running on that compute node. In this sectionwe are not going to monitor other resources as Images, Storage and Networking. On each Compute node we want metwer, install the component as reported [here](http://docs.openstack.org/liberty/install-guide-rdo/ceilometer-nova.html).

####Working with the metering service







    
