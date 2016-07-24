###Advanced Neutron Configuration
The basic Neutron configuration uses tenant networks to provide internet access to the instances. All traffic coming from the Compute nodes is routed through the Network node. In this sections we are going to enhance the basic scenario by introducing advanced features of the Neutron Service.

####Flat Provider network scenario
Provider networks are created by the OpenStack administrator and map directly to an existing physical network in the data center. Use flat provider networks to connect instances directly to the external network. 

On the Controller node, edit the ``/etc/neutron/plugin.ini`` initialization file

```
[ml2]
type_drivers = vxlan,flat

[ml2_type_flat]
flat_networks = *
```

Restart the Neutron service to apply the change
```
systemctl restart neutron-server
```

On the Network node, 





####VLAN based Provider network scenario
