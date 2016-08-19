###Working with Heat Templates
In addition to the AWS CloudFormation templates, the OpenbStack Heat Orchestration service uses the Heat OpenStack Templates (**HOT**) written in **YAML**. YAML stands for “Yet Another Markup Language” and is refreshingly easy to read and understand by everyone.

Some basic terminology is in order to help navigate the YAML structure:

   1.**Stack**: this is what we are creating: a collection of VMs and their associated configuration. In Heat, the term “Stack” has a very specific meaning. It refers to a collection of resources. Resources could be instances, networks, security groups, and even auto-scaling rules.
   2.**Template**: this is the design-time definition of the resources that will make up the stack. For example, if we want to have two instances connected on a private network, we need to define a template for each instance as well as the network. A template is made up of four different sections:
   
    * **Resources**: these are the details of the specific stack. These are the objects that will be created or modified when the template runs. Resources could be Instances, Volumes, Security Groups, Floating IPs, or any number of objects in OpenStack.
    * **Properties**: these are specifics of the template. For example, if we want to specify a CentOS instance in a small flavor. Properties may be hard coded in the template, or may be prompted as parameters.
    * **Parameters**: these are properties values that must be passed when running the Heat template. In HOT format, they appear before the resources section and are mapped to properties.
    * **Output**: this is what is passed back to the user. It may be displayed in the dashboard, or revealed in command line ``heat stack-list`` or ``heat stack-show`` commands.
