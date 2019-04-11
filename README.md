# HBA and Multipath Driver Configuration

In recent years, the popularity of Cloud Service Providers such as Amazon Web Services, Microsoft Azure and Google Cloud have decreased the need to utilize on-premise (on-prem) systems and storage.  However, cloud technology has not eliminated the use of on-prem.  In some cases, businesses may not be able to use a third party provider which would require them to manage their own data and services in their own datacenter.

Businesses will need to provide their own storage services when managing their own datacenter.  Storage can be "presented" to systems utilizing different technologies (i.e., fibre channel, iSCSI, NFS, CIFS).  This tutorial is written specifically for fibre channel storage configuration on a RHEL host.  Assumptions are that the RHEL system has a Host Bus Adapter (HBA) or multiple HBAs installed and that the system has multiple data paths configured to the storage system. 

## HBA Configuration

Storage and system utilities must be installed to view HBA status and disk targets.  If the following utilties are not installed, run the following command:
```
yum install sysfsutils sg3-utils lsscsi -y
```
lspci is a utility for displaying information about PCI buses in the system and devices connected to them. The command can be used to list HBAs installed on a system:
```
lspci -nn | grep -i HBA
```
This example shows that two QLogic HBAs are installed:
```
15:00.0 Fibre Channel: QLogic Corp. ISP2532-based 8Gb Fibre Channel to PCI Express HBA (rev 02)
15:00.1 Fibre Channel: QLogic Corp. ISP2532-based 8Gb Fibre Channel to PCI Express HBA (rev 02)
```
systool is a utility that can be used to view system device information by bus, class, and topology.  It can be used to view additional HBA information, specifically the World Wide Names (WWNs) of the HBAs which are required when connecting a RHEL host to a Storage Area Network (SAN) device.  The following command will display the WWNs:
```
systool -c fc_host -A port_name
```
The output of the command will be similar to the following:
```
Class = "fc_host"
  Class Device = "host5"
    port_name           = "0x21000024ff3434e4"
    Device = "host5"
  Class Device = "host6"
    port_name           = "0x21000024ff3434e5"
    Device = "host6"
```
The values for "port_name" can be provided to a SAN administator.  The SAN administrator will use those values to configure the RHEL host to an appropriate storage array for LUN access.

The lsscsi utility can be used to list SCSI devices (or hosts) and their attributes.  Once a LUN has been presented to the RHEL host, run the following command to view the storage devices attatched to the system:
```
lsscsi -k
```
Devices may be displayed as follows:
```
[0:0:0:0]    disk    ServeRA  8k-l Mirror      V1.0  /dev/sda 
[0:1:0:0]    disk    IBM-ESXS GNA300C3ESTT0Z N BH0G  -       
[0:1:1:0]    disk    IBM-ESXS ST3300555SS      BA33  -       
[0:3:0:0]    enclosu IBM-ESXS VSC7160          1.06  -       
[2:0:0:0]    cd/dvd  MATSHITA UJDA770 DVD/CDRW 1.21  /dev/sr0 
[3:0:0:1]    disk    IBM      DCS9550          4.03  /dev/sdb 
[3:0:0:2]    disk    IBM      DCS9550          4.03  /dev/sdc 
[3:0:0:3]    disk    IBM      DCS9550          4.03  /dev/sdd 
[3:0:0:4]    disk    IBM      DCS9550          4.03  /dev/sde 
[3:0:0:9]    disk    IBM      DCS9550          4.03  /dev/sdf 
[3:0:0:10]   disk    IBM      DCS9550          4.03  /dev/sdg
```
From the output, we can see that there are a number of paths ([3:0:0:x]) pointing to the same disk.  In this case, the system can be configured to utilize the multipathing driver which allows an administrator to logically consolidate the paths into a single instance.  

## Configure multipathing

The mulitpath driver can be installed using the following command:
```
yum install device-mapper-multipath -y
```
In some instances, the mulitpath configuration file (multipath.conf) is not setup.  If the configuration file is not setup, run the following command to copy a template of the multipath.conf file:
```
cp /usr/share/doc/device-mapper-multipath*/multipath.conf /etc
```
One advantage of utilizing the multipath driver is that it can be configured to allow an administrator to set their own use friendly names for disk targets.  This is set in defaults section of the multipath.conf file.  Uncomment the "user_friendly_names" and "path_grouping_policy" values.  Set them to the following:
```
user_friendly_names	yes
path_grouping_policy	multibus
```
Any changes to multipath.conf requires a restart of services:
```
systemctl restart multipathd
```
To display devices from the multipath driver perspective, run the following command:
```
multipath -ll
```
Example out is as follows:
```
mpathb (360014051f89d2bb3300470fa7d4baa10) dm-2 LIO-ORG ,lun0
size=2.0G features='0' hwhandler='0' wp=rw
|-+- policy='service-time 0' prio=0 status=active
| `- 1:0:0:0 sdb 8:16 active active running
`-+- policy='service-time 0' prio=0 status=enabled
  `- 2:0:0:0 sdc 8:32 active active running
```
This output shows that a LUN labeled as mpathb is configured as two paths (sdb and sdc).  The LUN is 2.0Gb in size.

The multipath driver can also be configured to utilize aliases.  This can be helpful when a system is configured with multiple LUNs.  Using the values from the previous output, an administrator can rename "mpathb" to "test-lun" by updating the multipath.conf file.  To do so, open multipath.conf and go the the multipaths section and update the following:
```
multipath {
              wwid                  360014051f89d2bb3300470fa7d4baa10
              alias                 test-lun
              rr_weight             priorities
```
Restart the mulitpath driver:
```
systemctl restart multipathd
```
Rerun the multipath command:
```
multipath -ll
```
This output should now display:
```
test-lun (360014051f89d2bb3300470fa7d4baa10) dm-2 LIO-ORG ,lun0
size=2.0G features='0' hwhandler='0' wp=rw
|-+- policy='service-time 0' prio=0 status=active
| `- 1:0:0:0 sdb 8:16 active active running
`-+- policy='service-time 0' prio=0 status=enabled
  `- 2:0:0:0 sdc 8:32 active active running
```

The LUN can now be referenced using its "test-lun" name.  An administrator can now add the LUN to LVM using the "test-lun" name/label and configure it for system use.
