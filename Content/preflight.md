###Initial server setup for an OpenStack deployment

The x86_64 platform is currently the only supported architecture. Machine with at least 4GB RAM, processors with hardware virtualization extensions, and one or more network adapters. Both Intel and AMD CPU support virtualization technology which allows multiple operating systems to run simultaneously on an x86 computer in a safe and efficient manner using hardware virtualization. Use the following commands to verify that if hardware virtualization extensions is enabled or not in your BIOS. On the compute nodes type the following commands as root to verify that host cpu has support for Intel VT or AMD-V technology.

```
# grep --color vmx /proc/cpuinfo
# grep --color svm /proc/cpuinfo
```

####Install CentOS 7 minimal
Install CentOS 7 minimal on the target servers.

Update the systems

``# yum update -y``

Since CentOS minimal comes without common tools installed, install them:

```
# yum install net-tools
# yum install bind-utils
```

Stop and disable the NetworkManager service, because we don’t need it on the server:
```
# systemctl stop NetworkManager 
# systemctl disable NetworkManager
```
Restart the network service

```
# systemctl restart network.service
```

Install, configure and start the NTP service
```
# yum install -y ntp
# systemctl enable ntpd.service
# systemctl start ntpd.service
# ntpq -c peers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ntp1.inrim.it   .CTD.            1 u    2   64    1   20.958    3.063   1.029
+ntp2.inrim.it   .CTD.            1 u    1   64    1   27.671    3.232   0.299
```

Setup the RDO repository

```
yum install -y https://rdoproject.org/repos/rdo-release.rpm

```
