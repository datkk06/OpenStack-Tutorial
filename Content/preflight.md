###Initial server setup for an OpenStack deployment

The x86_64 platform is currently the only supported architecture. Machine with at least 2GB RAM, processors with hardware virtualization extensions, and at least one network adapter. Both Intel and AMD CPU support virtualization technology which allows multiple operating systems to run simultaneously on an x86 computer in a safe and efficient manner using hardware virtualization. Use the following commands to verify that if hardware virtualization extensions is enabled or not in your BIOS.

####Verify CPU Virtualization Extensions
Type the following command as root to verify that host cpu has support for Intel VT technology.

``# grep --color vmx /proc/cpuinfo``

If the output has the ``vmx`` flags, then Intel CPU host is capable of running hardware virtualization.

Type the following command as root to verify that host cpu has support for AMD-V technology

``# grep --color svm /proc/cpuinfo``

Again, the output has the ``svm`` flags, then AND CPU host is capable of running hardware virtualization.

####Check BIOS Settings
Many, system manufacturers disable AMD or Intel virtualization technology in the BIOS by default. You need to reboot the system and turn it in the BIOS.

####Install CentOS 7 minimal
Install CentOS 7 minimal on the target server.

Update the system

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
