###Heat Orchestration Service
OpenStack Heat is the orchestration service that allow to spin up multiple instances, logical networks, and other cloud services in an automated fashion. Heat major components are:

*The **heat-api** component implements an OpenStack-native RESTful API. This components processes API requests by sending them to the Heat engine via AMQP.

*The **heat-api-cfn** component provides an API compatible with AWS CloudFormation, by forwarding API requests to the Heat engine over AMQP.

*The **heat-engine** component provides the main orchestration functionality.

All of these components would typically be installed on the Controller node even if there is nothing that requires them to be installed on the same system. Like other OpenStack services, Heat uses a back-end MySQL database for maintaining state information.
