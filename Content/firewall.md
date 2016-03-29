###Configure Firewall FWaaS Service
The Firewall as a Service (**FWaaS**) agent adds perimeter firewall management to OpenStack Networking. FWaaS uses iptables to apply firewall policy to all virtual routers within a project, and supports one firewall policy and logical firewall instance per project. FWaaS operates at the perimeter by filtering traffic at the Neutron router. This is different from Security Groups, which operate at the instance level.

