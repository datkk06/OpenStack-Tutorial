###Cloud Scaling
When a new OpenStack cloud is started, all servers run over the same hardware and all servers are in the same building, room, rack, even a chassis when the cloud is in the first growth paces. After a while, workloads increase and the current hardware set is not enough to process that workloads and new hardware is bought. This hardware has different storage disks, CPU, RAM and so on. Also racks, rooms and buildings are too small and the growing cloud needs redundancy between cities or regions.

OpenStack offers solutions for scaling the Cloud, called **Regions**, **Cells**, **Availability Zones** and **Host Aggregates**. Each method provides different functionality and can be best divided into two groups:

1. **Cells** and **Regions**: segregate an entire Cloud and result in running separate Compute deployments.
2. **Availability Zones** and **Host Aggregates**: divide a single Compute deployment.

**Cells** are still considered experimental.

**Regions** have a separate API endpoint per installation, allowing for a discrete separation of Compute deployments. Users wanting to run instances across sites have to explicitly select a region. Each region has a full nova installation, the only shared services are Keysone and Horizon. A different API endpoint exists for every region.

**Availability Zones** inside a region, represent a logical partition of the a single Compute deployment in the form of racks, rooms, buildings, etc. Users can run instances in the desired Availability Zone. Keystone and all nova services are shared between Availability Zones.

**Host Aggregates** represent a logical set of properties/characteristics a group of hosts owns in the form of metadata. For example, if some of servers have SSD and the other have SATA, you can map those properties to a group of hosts, when a image or flavor with the meta parameter associated is started, the Nova Scheduler will filter the available hosts with the meta parameter value and boot the instance on hosts with the desired property. Host Aggregates are managed by OpenStack admins only.


###Implementing Availability Zones and Host Aggregates
Considering an OpenStack Cloud made of a single Compute deployment. Default Availability Zone is the _nova_ zone.
