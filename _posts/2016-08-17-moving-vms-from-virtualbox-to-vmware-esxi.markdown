---
layout: post
title:  "Moving VMs from VirtualBox to VMware ESXi"
date:   2016-08-17 13:00:39 +0200
categories: sysadmin
---
Over the last few days I've been working on a workflow to move 
virtual machines from our development and testing environment 
which runs headless virtualbox to our production VMWare ESXi 5.1 
environment.

Here is the process that has worked for me. 

# Environment

1. Virtualbox server, `vboxserver.example.com`. One of the VM's 
is called `myvm`.

2. ESXi server, `esxiserver.example.com`. We use the free license 
running standalone servers without any virtual center.

3. Management workstation.

# Tools

1. `VBoxManage`, VirtualBoxes command line management tool.

2. `ovftool`, VMWare's OVF tool. It is free but you need to register
and log in to get it.

# Process

Export the virtualbox vm to a Open Virtualization Archive (OVA).
Run the following command on the virtualbox server:

    `VBoxManage export myvm -o myvm.ova`

On a host with `ovftool` installed convert the `ova` to an `ovf`.

    `ovftool  --lax ../myvm.ova myvm.ovf`

Edit the `myvm.ovf` configuration file and change some virtualhox config
to vmware config.

In the ovf file change the System Type from `virtualbox-2.2` to `vmx-07`.

```xml
<vssd:VirtualSystemType>virtualbox-2.2</vssd:VirtualSystemType>
```

```xml
<vssd:VirtualSystemType>vmx-07</vssd:VirtualSystemType>
```

Change the SATA controller to a SCSI controller.

```xml
<Item>
<rasd:Address>0</rasd:Address>
<rasd:Caption>sataController0</rasd:Caption>
<rasd:Description>SATA Controller</rasd:Description>
<rasd:ElementName>sataController0</rasd:ElementName>
<rasd:InstanceID>5</rasd:InstanceID>
<rasd:ResourceSubType>AHCI</rasd:ResourceSubType>
<rasd:ResourceType>20</rasd:ResourceType>
</Item>
```

to:

```xml
<Item>
<rasd:Address>0</rasd:Address>
<rasd:Caption>SCSIController</rasd:Caption>
<rasd:Description>SCSI Controller</rasd:Description>
<rasd:ElementName>SCSIController</rasd:ElementName>
<rasd:InstanceID>5</rasd:InstanceID>
<rasd:ResourceSubType>lsilogic</rasd:ResourceSubType>
<rasd:ResourceType>6</rasd:ResourceType>
</Item>
```

Since we changed `myvm.ovf` the checksum will fail. Delete
the myvm.mf will stop this being checked.

Push ovf file (and it's associated disk image) to ESXi server
using VMWare's `ovftool`.

`ovftool --net:"NAT=VM Network" -ds=datastore2 myvm.ovf vi://root@esxiserver.example.com`

```
Opening OVF source: myvm.ovf
Enter login information for target vi://esxiserver.example.com/
Username: root
Password: ********
Opening VI target: vi://root@esxiserver.example.com:443/
Deploying to VI: vi://root@esxiserver.example.com:443/
Transfer Completed                    
Warning:
 - No manifest file found.
 - No manifest entry found for: 'myvm-disk1.vmdk'.
Completed successfully
```

Log into the VMWare server and start the VM. 

If you are lucky everything will work :)

