###Ceilometer Metering Service
OpenStack Ceilometer services provides metering data related to OpenStack services. It collects events and metering data by monitoring notifications from the other service services and publishes collected data to various targets including data stores and message queues.

Ceilometer main components are:

1.  The **ceilometer-compute**, an agent running on each compute node polling resources for statistics.
2.  The **ceilometer-central**, an agent running on the other nodes, including the controller, polling for statistics not related to the compute resources.
3.  The **ceilometer-notification**, an agent running on the controller, responsible for consuming messages from the message queues and building events and metering data.
4.  The **ceilometer-collector**, running on the controller, dispatching collected telemetry data to a data store or external message queues without modification.
5.  The **ceilometer-alarm-evaluator**, running on the controller, responsible for evaluating when an alarm has to be fired.
6.  The **ceilometer-alarm-notifier**, running on the controller, responsible for notify alarms based on the threshold evaluation for a given set of samples.
7.  The **ceilometer-api**, running on the controller providing access to the users. Only the collector and API server have access to the data store.


