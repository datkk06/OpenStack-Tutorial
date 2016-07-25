###Open vSwitch Layout
The OpenStack framework is able to use different Software Defined Storage solutions. The default option is the Open vSwitch as per ``/etc/neutron/plugin.ini`` configuration file on the Controller node. The Open vSwitch installed on the Network node and all the Compute nodes is controlled by the Neutron service via the Open vSwitch Neutron Agents wich settings are in configuration file ``/etc/neutron/plugins/ml2/openvswitch_agent.ini``.

On the Controller node
```
# vi /etc/neutron/plugin.ini
[ml2]
# (ListOpt) Ordered list of networking mechanism driver entrypoints
# to be loaded from the neutron.ml2.mechanism_drivers namespace.
# mechanism_drivers = openvswitch
mechanism_drivers = openvswitch
# Example: mechanism_drivers = openvswitch,mlnx
# Example: mechanism_drivers = arista
# Example: mechanism_drivers = openvswitch,cisco_nexus,logger
# Example: mechanism_drivers = openvswitch,brocade
# Example: mechanism_drivers = linuxbridge,brocade
```

On the Network and all Compute nodes
```
# vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
tunnel_types = vlan, vxlan, gre
bridge_mappings = external:br-ex
...
```

The **Integration Bridge** acts as a virtual "patch bay". On Compute node, all VM virtual interfaces are attached to this bridge and then "patched" according to their network connectivity. On the Network node, all DHCP Server instances and L3 Router, Firewall and Load Balancer are attached to this bridge.

The **Tunnel Bridge** provides the connettivity for the tenant networks between Compute nodes and Compute nodes and Network node. The tunnel type can be VLAN, VxLAN or GRE based, depending on the setup.

Finally, the **External Bridge** provides a mapping between the physical network and the Open vSwitch layout inside both Compute and Network nodes, depending on the setup. In case of Tenant networks only scenario, the Compute nodes are not attached to the external network and the external bridge is not present on the layout. The external bridge is only present on the layout of the Network node. In case of the Provider network scenario, the external bridge will be present also on the Compute nodes since it provides connectivity to the outside via the provider networks.

Making changes to the Open vSwitch Agent configuration requires always the agent restart
```
# systemctl restart neutron-openvswitch-agent
```

####Tenant network scenario
We assume a Network node providing access to the outside and two Compute nodes for VMs. We'll check both the East-West traffic between VMs belonging to the same tenant and the North-South traffic between a VM and the outside. Tenant networks connectivity is based on VxLAN tunnels.

Starting with an empty layout without VMs. To check the layout created by Neutron use the ``ovs-vsctl show`` command.

On the Network node
```
# ovs-vsctl show
d7930874-e155-42d7-978a-f78d0bcb218e
    Bridge br-ex
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
        Port br-ex
            Interface br-ex
                type: internal
        Port "ens33"
            Interface "ens33"
    Bridge br-int
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
    Bridge br-tun
        fail_mode: secure
        Port br-tun
            Interface br-tun
                type: internal
        Port "vxlan-c0a80120"
            Interface "vxlan-c0a80120"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.38", out_key=flow, remote_ip="192.168.1.32"}
        Port "vxlan-c0a80122"
            Interface "vxlan-c0a80122"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.38", out_key=flow, remote_ip="192.168.1.34"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    ovs_version: "2.4.0"
```

On the Compute 01 node
```
# ovs-vsctl show
7bb6d324-ebd3-4c44-8448-63b6da83fc93
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-c0a80122"
            Interface "vxlan-c0a80122"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.32", out_key=flow, remote_ip="192.168.1.34"}
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-c0a80126"
            Interface "vxlan-c0a80126"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.32", out_key=flow, remote_ip="192.168.1.38"}
    Bridge br-int
        fail_mode: secure
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "2.4.0"
```

On the Compute 02 node
```
# ovs-vsctl show
e0ec8cf3-8df5-4ac5-9718-4223bcd54b0c
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-c0a80120"
            Interface "vxlan-c0a80120"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.34", out_key=flow, remote_ip="192.168.1.32"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
        Port "vxlan-c0a80126"
            Interface "vxlan-c0a80126"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="192.168.1.34", out_key=flow, remote_ip="192.168.1.38"}
    Bridge br-int
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
    ovs_version: "2.4.0"
```

