###Working with Heat Templates
In addition to the AWS CloudFormation templates, the OpenbStack Heat Orchestration service uses the Heat OpenStack Templates (**HOT**) written in **YAML**. YAML stands for “Yet Another Markup Language” and is refreshingly easy to read and understand by everyone.

Some basic terminology is in order to help navigate the YAML structure:

1. **Stack**: this is what we are creating: a collection of VMs and their associated configuration. In Heat, the term “Stack” has a very specific meaning. It refers to a collection of resources. Resources could be instances, networks, security groups, and even auto-scaling rules.

2. **Template**: this is the design of the resources that will make up the stack. For example, if we want to have two instances connected on a private network, we need to define a template for each instance as well as the network. A template is made up of four different sections:
  * **Resources**: these are the details of the specific stack. These are the objects that will be created or modified when the template runs. Resources could be Instances, Volumes, Security Groups, Floating IPs, or any number of objects in OpenStack.
  * **Properties**: these are specifics of the template. For example, if we want to specify a CentOS instance in a small flavor, we need to define a property for the "CentOS" image and a property for the "small" flavor. Properties may be hard coded in the template, or may be prompted as parameters.
  * **Parameters**: these are properties values that must be passed when running the Heat template. In HOT format, they appear before the resources section and are mapped to properties.
  * **Output**: this is what is passed back to the user. It may be displayed in the dashboard, or revealed in command line ``heat stack-show`` commands.

####A very basic example
In this section we are going to implement a simple HOT template example for starting an instance. This is not so useful but it helps to understand the basics. Here the example:

```
# vi heat-template-01.yaml
heat_template_version: 2015-10-15
  description: Simple template to deploy a single compute instance
  
  resources:
    my_instance:
      type: OS::Nova::Server
      properties:
        image: cirros
        flavor: small
        key_name: demokey
        networks:
          - network: tenant_network
```

First, each template has to include a valid version and an optional description telling the user what the template is doing

    heat_template_version: 2015-10-15
      description: Simple template to deploy a single compute instance


In the **resources** section, there is just one type of resource: a server. We know it is a server because its type tells us it is an ``OpenStack Nova Server (OS::Nova::Server)``. We have given it a name: *“my_instance”*.

We want our server to have certain properties:
* image: this instance is based on a certain image that we have in our OpenStack glance repository
* flavor: it uses certain size or flavor
* key-name: we also want to control access to this instance by injecting a keyname
* network: the instance has to be placed on a specific network

To create the stack, make sure the user has the Heat Stack Owner role
```
# source keystonerc_admin
# openstack role list --user demo --project demo
+----------------------------------+------------------+---------+------+
| ID                               | Name             | Project | User |
+----------------------------------+------------------+---------+------+
| 9fe2ff9ee4384b1894a90878d3e92bab | _member_         | demo    | demo |
| be2de476946c43dfb71196a961eebc6a | heat_stack_owner | demo    | demo |
+----------------------------------+------------------+---------+------+
```

Login as user to the Controller node and run the stack creation
```
# source keystonerc_demo
# heat stack-create first_heat_stack -f first_heat_stack.yaml
# heat stack-list
+--------------------------------------+------------------+-----------------+---------------------+--------------+
| id                                   | stack_name       | stack_status    | creation_time       | updated_time |
+--------------------------------------+------------------+-----------------+---------------------+--------------+
| b546cd6d-38b6-4ef0-91fa-54314093e305 | first_heat_stack | CREATE_COMPLETE | 2016-08-19T13:34:48 | None         |
+--------------------------------------+------------------+-----------------+---------------------+--------------+
```

After stack creation completes, check the instance created
```
# nova list --fields name
+--------------------------------------+-------------------------------------------+
| ID                                   | Name                                      |
+--------------------------------------+-------------------------------------------+
| c58fc820-a2a4-403e-b128-c473d960856b | first_heat_stack-my_instance-7fnnigpd6s3b |
+--------------------------------------+-------------------------------------------+
```

The stack can be deleted including all resources created
```
# heat stack-delete first_heat_stack
# nova list --fields name
+--------------------------------------+-------+
| ID                                   | Name  |
+--------------------------------------+-------+
+--------------------------------------+-------+
```

####Parameters Stack example
In the above example, all the properties are hardcoded into the **resources** section. Alternatively, we can specify those values as input parameters in the **parameters** section and ask the user to provide the values. In this way, the user can start different stacks without change the template file

```
# vi second_heat_stack.yaml
heat_template_version: 2015-10-15
description: Simple template to deploy a single compute instance

parameters:
  image:
    type: string
    label: Image name or ID
    description: Image to be used for compute instance
  flavor:
    type: string
    label: Flavor
    description: Type of instance (flavor) to be used
  key:
    type: string
    label: Key name
    description: Name of key-pair to be used for compute instance
  private_network:
    type: string
    label: Private network name or ID
    description: Network to attach instance to.
    default: tenant-network

resources:
  my_instance:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      key_name: { get_param: key }
      flavor: { get_param: flavor }
      networks:
        - network: { get_param: private_network }

 outputs:
  instance_name:
    description: Name of the instance
    value: { get_attr: [my_instance, name] }
  instance_ip:
    description: IP address of the instance
    value: { get_attr: [my_instance, first_address] }
```

Login as user to the Controller node and run the stack creation by providing the requested input parameters
```
# source keystonerc_demo
# heat stack-create second_heat_stack -f second_heat_stack.yaml
ERROR: The Parameter (image) was not provided.

# heat stack-create second_heat_stack \
-f second_heat_stack.yaml \
-P "image=cirros;key=demokey;flavor=small;private_network=tenant-network"

# heat stack-list
+--------------------------------------+-------------------+-----------------+---------------------+--------------+
| id                                   | stack_name        | stack_status    | creation_time       | updated_time |
+--------------------------------------+-------------------+-----------------+---------------------+--------------+
| 1d1f4824-91a3-48c4-9828-b9fbf49e28f8 | second_heat_stack | CREATE_COMPLETE | 2016-08-19T14:03:42 | None         |
+--------------------------------------+-------------------+-----------------+---------------------+--------------+
```

We can also define default values for input parameters which will be used in case the user does not provide the parameter during deployment. In addition, we can restrict the imput value to a specific set of predefined values. For example, the following definition for the flavor parameter will select the *"small"* flavor unless specified otherwise by the user. Also the user is forced to choise between a predefined list of flavors
```
parameters:
  flavor:
    type: string
    label: Flavor
    description: Type of instance (flavor) to be used
    constraints:
    - allowed_values: [small, medium, large]
    default: small
```


In the **output** section, we specified outputs to the user: the name of the instance and the IP address. Otherwise, the user would have to look it up themselves once the stack has been created.
```
 outputs:
  instance_name:
    description: Name of the instance
    value: { get_attr: [my_instance, name] }
  instance_ip:
    description: IP address of the instance
    value: { get_attr: [my_instance, first_address] }
```

After stack creation, check the output from the template
```
# heat stack-show second_heat_stack | grep output
|     "output_value": "second_heat_stack-my_instance-5q5knjktvy5b",                                           
|     "output_key": "instance_name"                                                                           
|     "output_value": "192.168.1.25",                                                                         
|     "output_key": "instance_ip"                                                                             
```






