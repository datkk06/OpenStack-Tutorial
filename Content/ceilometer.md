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
In this section we are going to work with few basic concepts of the metering service:

* [Meters](./ceilometer.md#meters)
* [Samples](./ceilometer.md#samples)
* [Statistics](./ceilometer.md#statistics)
* [Alarms](./ceilometer.md#alarms)
* [Pipelines](./ceilometer.md#pipelines)

##### Meters
A meters measures a particular aspect of resource usage as for example, the current CPU utilization % for an instance. The lifecycle of meters is decoupled from the existence of the related resources, in the sense that the meter continues to exist after the resource has been terminated. All meters have a string name, an unit of measurement, and a type indicating whether values are monotonically increasing (i.e. *cumulative*), interpreted as a change from the previous value (i.e. *delta*), or a standalone value relating only to the current duration (i.e. *gauge*).

The following returns all meters available on the system

    # ceilometer meter-list
    +---------------------------------+------------+-----------+------------------------------------------
    | Name                            | Type       | Unit      | Resource ID
    +---------------------------------+------------+-----------+------------------------------------------
    | cpu                             | cumulative | ns        | 9d29e3bd-caa1-4c04-b570-9dec555c8adc
    | cpu.delta                       | delta      | ns        | 9d29e3bd-caa1-4c04-b570-9dec555c8adc
    | cpu_util                        | gauge      | %         | 9d29e3bd-caa1-4c04-b570-9dec555c8adc
    | disk.allocation                 | gauge      | B         | 9d29e3bd-caa1-4c04-b570-9dec555c8adc
    | disk.capacity                   | gauge      | B         | 9d29e3bd-caa1-4c04-b570-9dec555c8adc 
    ...

To query for a specific resource or project or user

    # ceilometer meter-list --query resource=<resource_id>
    # ceilometer meter-list --query project=<project_id>
    # ceilometer meter-list --query user=<user_id>

##### Samples
A sample is the individual datapoints associated with a particular meter. As such, all samples encompass the same attributes as the meter itself, but with the addition of a timestamp and and a value, also called the sample volume.

Here to show last 10 samples associated with a given meter

    # ceilometer sample-list --meter cpu_util --limit 10
    +--------------------------------------+----------+-------+---------------+------+----------------------------+
    | Resource ID                          | Name     | Type  | Volume        | Unit | Timestamp                  |
    +--------------------------------------+----------+-------+---------------+------+----------------------------+
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.5923854989 | %    | 2016-08-23T17:57:27.681000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6333676252 | %    | 2016-08-23T17:56:27.922000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6422183317 | %    | 2016-08-23T17:55:27.687000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6170192908 | %    | 2016-08-23T17:54:27.682000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.5867382656 | %    | 2016-08-23T17:53:27.683000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6299362877 | %    | 2016-08-23T17:52:27.920000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6039615059 | %    | 2016-08-23T17:51:27.822000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6117520497 | %    | 2016-08-23T17:50:27.674000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6477669865 | %    | 2016-08-23T17:49:27.671000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.9214347448 | %    | 2016-08-23T17:48:27.659000 |
    +--------------------------------------+----------+-------+---------------+------+----------------------------+

##### Statistics
##### Alarms
##### Pipelines





    
