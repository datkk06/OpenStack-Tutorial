###Load Balancer Configuration
A Load Balancer enables networking to distribute incoming requests evenly among designated virtual machines instances. This distribution ensures that the workload is shared predictably among instances and enables more effective use of system resources. Load Balancers use one of these balancing methods to distribute incoming requests:

|Method|Description|
|------|-----------|
|Round Robin|Rotates requests evenly between multiple instances|
|Source IP|Requests from a unique source IP address are consistently directed to the same instance|
|Least connections|Allocates requests to the instance with the least number of active connections|

In this section, we are going to configure a simple load balancer to balance incoming traffic toward a couple of virtual machines running a web server. Both Load Balancer V1 and V2 are available. Version 1 will be replaced by Version 2 but it is still available. At time of writing, the Version 2 is not yet supported via Horizon GUI.

####Load Balancer Version V1
To configure the Load Balancer service, on the Network node, edit the LBaaS Agent by editing the ``/etc/neutron/lbaas_agent.ini`` initialization file
```
[DEFAULT]
debug = True
periodic_interval = 10
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
device_driver = neutron.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver

[haproxy]
loadbalancer_state_path = $state_path/lbaas
user_group = haproxy
send_gratuitous_arp = 3
```

Start and enable the service on the Network node
```
# systemctl start neutron-lbaas-agent
# systemctl enable neutron-lbaas-agent
```

As tenant user, create a couple of small virtual machine running on the tenant network
```
# nova boot \
--image $(nova image-list | awk '/ubuntu/ {print $2}') \
--flavor $(nova flavor-list | awk '/small/ {print $2}') \
--nic net-id=$(neutron net-list | awk '/tenant/ {print $2}') \
--key-name mytenant \
--security-groups myaccess \
VM01

# nova boot \
--image $(nova image-list | awk '/ubuntu/ {print $2}') \
--flavor $(nova flavor-list | awk '/small/ {print $2}') \
--nic net-id=$(neutron net-list | awk '/tenant/ {print $2}') \
--key-name mytenant \
--security-groups myaccess \
VM02

# nova list
+--------------------------------------+------+--------+------------+-------------+-----------------------------+
| ID                                   | Name | Status | Task State | Power State | Networks                    |
+--------------------------------------+------+--------+------------+-------------+-----------------------------+
| 1ee95232-e527-47d9-ab7e-df1a675704e5 | VM01 | ACTIVE | -          | Running     | tenant-network=192.168.1.13 |
| f0f78f85-a6c3-4847-b497-5130bc981194 | VM02 | ACTIVE | -          | Running     | tenant-network=192.168.1.14 |
+--------------------------------------+------+--------+------------+-------------+-----------------------------+
```

Enable security group ingress access for HTTP protocol
```
# neutron security-group-rule-create \
--protocol tcp \
--port-range-min 80 \
--port-range-max 80 \
--direction ingress \
myaccess
```

On each instance, login via SSH and start a simple HTTP service
```
ubuntu@vm01:~$ MYIP=$(ifconfig eth0|grep 'inet addr'|awk -F: '{print $2}'| awk '{print $1}'
ubuntu@vm01:~$ while true; do echo -e "HTTP/1.0 200 OK\r\n\r\nWelcome to $MYIP" | sudo nc -l -p 80 ; done &

ubuntu@vm02:~$ MYIP=$(ifconfig eth0|grep 'inet addr'|awk -F: '{print $2}'| awk '{print $1}'
ubuntu@vm02:~$ while true; do echo -e "HTTP/1.0 200 OK\r\n\r\nWelcome to $MYIP" | sudo nc -l -p 80 ; done &
```

As tenant user, create a load balancer pool based on Round Robin method and HTTP protocol
```
# neutron lb-pool-create \
--lb-method ROUND_ROBIN \
--protocol HTTP \
--subnet-id tenant-subnetwork \
--name my-lb-pool
```

Add the two Virtual Machines created above as memenbers to the Load Balancer pool
```
# neutron lb-member-create \
--weight 1 \
--address 192.168.1.17 \
--protocol-port 80 \
my-lb-pool

# neutron lb-member-create \
--weight 1 \
--address 192.168.1.18 \
--protocol-port 80 \
my-lb-pool

# neutron lb-member-list
+--------------------------------------+--------------+---------------+--------+----------------+--------+
| id                                   | address      | protocol_port | weight | admin_state_up | status |
+--------------------------------------+--------------+---------------+--------+----------------+--------+
| a0828bd5-da97-45b1-b4d4-8fef63acbf1b | 192.168.1.17 |            80 |      1 | True           | ACTIVE |
| ebd9e210-df12-4e64-bf2a-00ac57449200 | 192.168.1.18 |            80 |      1 | True           | ACTIVE |
+--------------------------------------+--------------+---------------+--------+----------------+--------+
```

Create a Virtual IP and associate to the pool. The Virtual IP will be the IP address of the Load Balancer
```
neutron lb-vip-create \
--name LB-Virtual-IP \
--address 192.168.1.250 \
--protocol HTTP \
--protocol-port 80 \
--subnet-id $(neutron subnet-list | awk '/tenant/ {print $2}') \
my-lb-pool
```

At this point the Load Balancer has been successfully created and should be functional. Traffic sent to address **192.168.1.250** on port 80 will be load-balanced across all active members of the pool, i.e. **192.168.1.17** and **192.168.1.18**. To make the load balancer externally accessible, create a floating IP address and associate it with the Virtual IP address.
```
# neutron floatingip-create external-flat-network
Created a new floatingip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| fixed_ip_address    |                                      |
| floating_ip_address | 172.16.1.209                         |
| floating_network_id | 0588949f-7a2a-43cc-a879-2ddbabea0d4a |
| id                  | 0f16047b-bc77-472c-b6f4-d62dc7993adb |
| status              | DOWN                                 |
| tenant_id           | 22bdc5a0210e4a96add0cea90a6137ed     |
+---------------------+--------------------------------------+

# neutron lb-vip-show LB-Virtual-IP
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| address             | 192.168.1.250                        |
| admin_state_up      | True                                 |
| id                  | 21e16c3a-cd16-4666-a851-023d0407d53e |
| name                | LB-Virtual-IP                        |
| pool_id             | 32dc27ea-ed0d-4a7e-8dde-734b210f50ca |
| port_id             | e9afb9b3-ab00-4188-921d-6c676fef6c12 |
| protocol            | HTTP                                 |
| protocol_port       | 80                                   |
| session_persistence |                                      |
| status              | ACTIVE                               |
+---------------------+--------------------------------------+

# neutron floatingip-associate 0f16047b-bc77-472c-b6f4-d62dc7993adb e9afb9b3-ab00-4188-921d-6c676fef6c12
```

The next step is to create a health monitor and associate it with the pool. The health monitor is responsible for periodically checking the health of each member of the pool.
```
# neutron lb-healthmonitor-create --delay 5 --type HTTP --max-retries 3 --timeout 2
Created a new health_monitor:
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| admin_state_up | True                                 |
| delay          | 5                                    |
| expected_codes | 200                                  |
| http_method    | GET                                  |
| id             | 0947aaf6-5d24-4e83-a053-1a22517738bb |
| max_retries    | 3                                    |
| pools          |                                      |
| tenant_id      | 22bdc5a0210e4a96add0cea90a6137ed     |
| timeout        | 2                                    |
| type           | HTTP                                 |
| url_path       | /                                    |
+----------------+--------------------------------------+
```

The above health monitor will perform an HTTP GET of the root path. This health check expects an HTTP status of 200 in the response, and the connection must be established within 2 seconds. This check will be retried a maximum of 3 times before a member is determined to be failed.

Associate the healt monitor to the load balancer pool
```
# neutron lb-healthmonitor-associate 0947aaf6-5d24-4e83-a053-1a22517738bb my-lb-pool
Associated health monitor 0947aaf6-5d24-4e83-a053-1a22517738bb
```

Now the Load Balancer setup is fully working. Incoming requests to the floating **172.16.1.209** will be translated to the fixed virtual address **192.168.1.250** and then forwarded in a Round Robin fashion to the Virtual Machines instances with **192.168.1.17** and **192.168.1.18**.
