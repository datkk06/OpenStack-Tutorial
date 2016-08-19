###Working with Heat Templates
In addition to the AWS CloudFormation templates, the OpenbStack Heat Orchestration service uses the Heat OpenStack Templates (**HOT**) written in **YAML**. YAML stands for “Yet Another Markup Language” and is refreshingly easy to read and understand by everyone.

Some basic terminology is in order to help navigate the YAML structure:

1. **Stack**: this is what we are creating: a collection of VMs and their associated configuration. In Heat, the term “Stack” has a very specific meaning. It refers to a collection of resources. Resources could be instances, networks, security groups, and even auto-scaling rules.

2. **Template**: this is the design of the resources that will make up the stack. For example, if we want to have two instances connected on a private network, we need to define a template for each instance as well as the network. A template is made up of four different sections:
  * **Resources**: these are the details of the specific stack. These are the objects that will be created or modified when the template runs. Resources could be Instances, Volumes, Security Groups, Floating IPs, or any number of objects in OpenStack.
  * **Properties**: these are specifics of the template. For example, if we want to specify a CentOS instance in a small flavor, we need to define a property for the "CentOS" image and a property for the "small" flavor. Properties may be hard coded in the template, or may be prompted as parameters.
  * **Parameters**: these are properties values that must be passed when running the Heat template. In HOT format, they appear before the resources section and are mapped to properties.
  * **Output**: this is what is passed back to the user. It may be displayed in the dashboard, or revealed in command line ``heat stack-list`` or ``heat stack-show`` commands.

####First eaxample
In this section we are going to implement a simple HOT template for starting an instance. This is not so useful but it helps to understand the basics. Here the example:

```
heat_template_version: 2016-08-19
  description: Simple template to deploy a single compute instance
  
  parameters:
    image:
      type: string
      label: Image name or ID
      description: Image to be used for compute instance
      default: cirros
    flavor:
      type: string
      label: Flavor
      description: Type of instance (flavor) to be used
      constraints:
      - allowed_values: [tiny, medium, small]
      default: small
    key:
      type: string
      label: Key name
      description: Name of key-pair to be used for compute instance
      default: demokey
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
        flavor: { get_param: flavor }
        key_name: { get_param: key }
        networks:
          - network: { get_param: private_network }
  
  outputs:
    instance_ip:
      description: IP address of the instance
      value: { get_attr: [my_instance, first_address] }
```

