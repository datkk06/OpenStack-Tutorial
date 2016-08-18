###High Availability
OpenStack framework currently meets high availability requirements for its own infrastructure services like MySQL backend database, RabbitMQ messaging server, Neutron services, Nova services, Cinder Services, etc. However, OpenStack does not guarantee high availability for individual guest instances. This section reports some common methods of implementing highly available systems, with an emphasis on the core OpenStack services and other open source services that are closely aligned with the OpenStack framework.

