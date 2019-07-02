---
layout: post
title:  "QEMU virtio-scsi VPD emulation"
description: Enhancing QEMU virtio-scsi with Block Limits vital product data (VPD) emulation
date:   2019-07-01
comments: true
tags: qemu scsi virtio-scsi
categories: article
---

{% include tags.html %}


Cloud providers must support a great variety of hardware to support the customer needs. This includes hardware that might not behave as the back-end virtualization technology expects it to, sometimes leading to problems in the created virtual machine (VM) or guest.

QEMU allows the guests to use any Small Computer System Interface (SCSI) storage of the host by using two distinct paravirtualization backends that provides what it is called SCSI pass-through. The first one is virtio-blk. In this mode, available in all hypervisors, the guest uses the real storage device as a block device just to read and write data, while QEMU emulates the SCSI device internally. When running in Linux hypervisors, QEMU offers a second back end called virtio-scsi. Using this back end, QEMU proxies the SCSI communication back and forth between the guest device and the physical device, allowing the guest device to use all advanced features that the real device might implement. The gains of using virtio-scsi comes at a cost: if the physical SCSI device misbehaves or QEMU does not handle it properly, the guest is directly affected.

There is an instance in which virtio-scsi does not handle the physical SCSI device gracefully, impacting the guest. During the device setup, QEMU uses an optional SCSI feature called vital product data (VPD) Block Limits to inform the guest of the transfer limits of the physical device. Because this feature is optional, the device might not implement it. In this case, QEMU does not provide an alternative way to pass the device transfer limits to the guest, which ends up setting the transfer limit to a default value.

If this default value conflicts with the actual transfer limits of the real device, the guest is unable to use it. It will send read/write commands that are larger than what the physical device could handle, causing SCSI errors in the hypervisor that will propagate back to the guest.

In this article, we’ll talk more about this QEMU virtio-scsi behavior in these scenarios and how the author was able to solve it. The following section covers some basics of the SCSI standard that is necessary to understand the problem domain. The “QEMU virtio-blk/virtio-scsi with SCSI pass-through” section elaborates on how QEMU and virtio-scsi taps into the SCSI communication between the guest kernel and the physical device to allow the guest to properly configure it. “The Block Limits problem with virtio-scsi” section explains the problem and the possible workarounds for it. In “A solution using emulation” section we detail how the author solved the problem by using the existing emulation of QEMU virtio-blk back end.


## QEMU and virtio-scsi with SCSI pass-through

SCSI is a series of standards that defines commands, protocols, and interfaces to connect and transfer data between computers and peripherals. For the purpose of this article, we’re going to detail the Inquiry command only.

We’ll also demonstrate the Inquiry command in practice with examples using the sg3_utils toolkit. This is a common package in most of the Linux distributions that allows the user to send and receive SCSI messages from user space to the device. In Fedora Linux, this can be installed with:

```code
sudo dnf install sg3_utils
```

### SCSI Inquiry command

The Inquiry command is used to request information from the SCSI device about its capabilities. On Linux, the kernel sends an Inquiry command for each detected SCSI device to query its attributes and set up them during the boot process.

![Figure1](/assets/images/qemu_SCSI_VPD1.png)
{: style="text-align: center"}

The enable vital product data (EVPD) bit determines whether this is an Inquiry for a specific VPD page or a standard Inquiry.

If EVPD is set to zero, then a standard Inquiry data is expected. The standard Inquiry data contains information such as vendor, peripherical device type, product identification, serial number, and so on.

Let’s use sg_inq, a command from sg3_utils, to issue a standard Inquiry to a real device. Let us consider a system with the following SCSI devices:

```code
$ lsscsi
[1:0:0:0]    disk    ATA      LITEON LCH-256V2 901   /dev/sda
```

We have a single Serial Advanced Technology Attachment (SATA) drive at /dev/sda. You can use the sq_inq command to send a standard Inquiry to it:

```code
$ sudo sg_inq /dev/sda
standard INQUIRY:
  PQual=0  Device_type=0  RMB=0  LU_CONG=0  version=0x05  [SPC-3]
  [AERC=0]  [TrmTsk=0]  NormACA=0  HiSUP=0  Resp_data_format=2
  SCCS=0  ACC=0  TPGS=0  3PC=0  Protect=0  [BQue=0]
  EncServ=0  MultiP=0  [MChngr=0]  [ACKREQQ=0]  Addr16=0
  [RelAdr=0]  WBus16=0  Sync=0  [Linked=0]  [TranDis=0]  CmdQue=1
  [SPI: Clocking=0x0  QAS=0  IUS=0]
    length=96 (0x60)   Peripheral device type: disk
 Vendor identification: ATA
 Product identification: LITEON LCH-256V2
 Product revision level: 901
 Unit serial number: SS0L68721L1BR6B104J6
```

As we can see, general information and capabilities about the device is retrieved in the response.

If the EVPD bit of the Inquiry command is set to one, then the PAGE_CODE byte contains the requested VPD page. A common use is to first request the supported VPD page (page code 00h) to determine what VPD pages the device supports. Then, issue an Inquiry request for each one of them.

In the same device we used before, we can retrieve the supported VPD pages data by using the sq_inq command with extra parameters:

```code
$ sudo sg_inq /dev/sda --vpd --page=0x00
VPD INQUIRY: Supported VPD pages page
Supported VPD pages:
0x0 Supported VPD pages
    0x80 Unit serial number
    0x83 Device identification
    0x89 ATA information
    0xb0 Block limits (sbc2)
    0xb1 Block device characteristics (sbc3)
    0xb2 Logical block provisioning (sbc3)
```

The --vpd parameter sets the EVPD bit to one, the --page parameter allows us to specify the VPD page we want.

Instead of using sq_inq and setting the EVPD manually, we can use another sg3_utils utility called sg_vpd that sets it automatically for us. It also has the advantage of being able to decode all VPD pages with one command.

Using sg_vpd to retrieve the ATA information page:

```code
$ sudo sg_vpd --page=0x89 /dev/sda
ATA information VPD page:
SAT Vendor identification: linux
SAT Product identification: libata
SAT Product revision level: 3.00
  Device signature indicates SATA transport
  ATA command IDENTIFY DEVICE response summary:
    model: LITEON LCH-256V2S
    serial number: SS0L68721L1BR6B104J6
    firmware revision: WC87901
```

There are several VPD pages defined in the SCSI spec, but two of them are mandatory: page 00h (supported VPD pages) and page 83h (device identification). Any device that complies with the spec must implement at least these two VPD pages. Most devices implement more than these two pages, especially the unit serial number (80h).


## QEMU virtio-blk/virtio-scsi with SCSI pass-through

To provide a virtual SCSI device, QEMU will either emulate the SCSI target and just read/write in the physical block device (virtio-blk back end), or it will pass-through the SCSI messages from the virtual machine kernel directly to the real device (virtio-scsi back end). The usability problem that we want to discuss happened with virtio-scsi, thus let’s elaborate on the basic functioning of it. But to get started, we’ll briefly explain how the virtio-blk back end works for two reasons: it makes easier to understand virtio-scsi and the mechanics of virtio-blk was part of the solution we’ll discuss in “A solution using emulation” section.


### virtio-blk

Simply put, this is a fully emulated back end as far as SCSI communication is concerned. The figure below illustrates how it operates.

![Figure2](/assets/images/qemu_SCSI_VPD2.png)
{: style="text-align: center"}

All SCSI commands responses are emulated in QEMU. For the read/write SCSI commands, QEMU will read/write the contents in the device block file in the hypervisor, emulating the SCSI reply back to the guest kernel.

we’ll now use the environment in which the virtio-scsi bug was found and fixed, which is an IBM POWER9 processor-based server with a 1.8 terabyte (TB) LSI MegaRAID 9361-8i storage. The QEMU command line to use a SCSI device using virtio-blk is:

```code
sudo qemu-system-ppc64 (...)
-device virtio-scsi-pci,id=scsi0,bus=pci.0,addr=0x2 \
-device scsi-hd,bus=scsi0.0,channel=0,scsi-id=0,lun=2,drive=drive-scsi0-0-0-2,id=scsi0-0-0-2 \
-drive file=/dev/disk/by-id/scsi-3600605b000a2c110ff0004053d84a61b,format=raw,if=none,id=drive-scsi0-0-0-2,cache=none
```

When using Libvirt, this is the element that must be added in the guest XML:

```xml
<disk type='block' device='disk'>
    <driver name='qemu' type='raw' cache='none'/>
    <source dev='/dev/disk/by-id/scsi-3600605b000a2c110ff0004053d84a61b'/>
    <target dev='sdc' bus='scsi'/>
    <address type='drive' controller='0' bus='0' target='0' unit='2'/>
</disk>
```

The /dev/disk/by/id/scsi-<id>(…) path points to a SCSI storage in the host:

```code
$ ls -la /dev/disk/by-id/scsi-3600605b000a2c110ff0004053d84a61b
lrwxrwxrwx. 1 root root 9 Jun 22 16:52 /dev/disk/by-id/scsi-3600605b000a2c110ff0004053d84a61b -> ../../sdb

```

Refer to the following code for its SCSI Inquiry details:

```code
$ lsscsi
(...)
[1:2:0:0]    disk    LSI      MR9361-8i        4.23  /dev/sdb
$ sudo sg_inq /dev/sdb
standard INQUIRY:
  PQual=0  Device_type=0  RMB=0  version=0x05  [SPC-3]
  [AERC=0]  [TrmTsk=0]  NormACA=0  HiSUP=0  Resp_data_format=2
  SCCS=0  ACC=0  TPGS=0  3PC=0  Protect=0  [BQue=0]
  EncServ=0  MultiP=0  [MChngr=0]  [ACKREQQ=0]  Addr16=0
  [RelAdr=0]  WBus16=0  Sync=0  Linked=0  [TranDis=0]  CmdQue=1
  [SPI: Clocking=0x0  QAS=0  IUS=0]
    length=96 (0x60)   Peripheral device type: disk
 Vendor identification: LSI
 Product identification: MR9361-8i
 Product revision level: 4.23
 Unit serial number: 001ba6843d050400ff10c1a200b00506
￼￼Show more
```
However, inside the guest operational system, the device identifies itself as QEMU HARDDISK.

```code
# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb               8:16   0  1.8T  0 disk
(…)
# sudo sg_inq /dev/sdb
standard INQUIRY:
  PQual=0  Device_type=0  RMB=0  version=0x05  [SPC-3]
  [AERC=0]  [TrmTsk=0]  NormACA=0  HiSUP=1  Resp_data_format=2
  SCCS=0  ACC=0  TPGS=0  3PC=0  Protect=0  [BQue=0]
  EncServ=0  MultiP=0  [MChngr=0]  [ACKREQQ=0]  Addr16=0
  [RelAdr=0]  WBus16=0  Sync=1  Linked=0  [TranDis=0]  CmdQue=1
    length=36 (0x24)   Peripheral device type: disk
 Vendor identification: QEMU
 Product identification: QEMU HARDDISK
 Product revision level: 2.5+
￼￼Show more
```

This shows that QEMU emulates the SCSI layer for this device, presenting it as a QEMU hard disk. While emulating, QEMU will take into account, among other things, the current configuration of the SCSI device in the host. This ensures that the configuration of the emulated device that the guest will use is compatible with the configuration of the real device. One of those configurations is related to the max_sectors_kb Linux kernel parameter that QEMU set in the Block Limits VPD response.


### max_sectors_kb and Block Limits VPD

max_sectors_kb is described in the [kernel documentation](https://www.kernel.org/doc/Documentation/block/queue-sysfs.txt) as:

“This is the maximum number of kilobytes that the block layer will allow for a filesystem request. Must be smaller than or equal to the maximum size allowed by the hardware.”

The value can be retrieved by reading sysfs. In the case of the MegaRAID device in the host, the value is:

```code
$ cat /sys/block/sdb/queue/max_sectors_kb
256
```

This means that any process running in the host can’t send any read or write request that exceeds 256 kilobytes to the /dev/sdb device.

Because QEMU is a process in the host, this limitation also applies to the virtual machine that uses /dev/sdb with SCSI pass-through. If the virtual machine attempts to use a greater value, QEMU won’t be able to read/write the host block file, that is, the guest SCSI disk won’t be usable.

The max_sectors_kb value of the host is retrieved by QEMU using an input/output control (or ioctl) called BLKSECTGET. This ioctl receives a valid file descriptor and a pointer to an integer in which the result will be restored. For example:

```C
ioctl(fd, BLKSECTGET, &max_sectors)
```

This command fetches the max_sectors_kb value of the block device that the file descriptor (fd) uses and stores it in the max_sectors variable.

This value is then added to the SCSI Block Limits VPD response. Block Limits is an optional VPD page that provides operating parameters such as Maximum/Optimal Transfer Length, Prefetch Length, and others. If the SCSI device supports it, the kernel requests the Block Limits page to set up the device parameters. The max_sectors_kb parameter is related to the Maximum Transfer Length value of the Block Limits response.

Inside the guest, let’s use sg_vpd to see the supported VPD pages for the /dev/sdb emulated pass-through device:

```code
# sg_vpd /dev/sdb
Supported VPD pages VPD page:
  Supported VPD pages [sv]
  Device identification [di]
  Block limits (SBC) [bl]
  Block device characteristics (SBC) [bdc]
  Logical block provisioning (SBC) [lbpv]
```


And retrieve the reply for the Block Limits page:

```code
# sg_vpd --page=bl /dev/sdb
Block limits VPD page (SBC):
  Write same no zero (WSNZ): 1
  Maximum compare and write length: 0 blocks
  Optimal transfer length granularity: 0 blocks
  Maximum transfer length: 512 blocks
  Optimal transfer length: 0 blocks
  Maximum prefetch length: 0 blocks
  Maximum unmap LBA count: 2097152
  Maximum unmap block descriptor count: 255
  Optimal unmap granularity: 8
  Unmap granularity alignment valid: 0
  Unmap granularity alignment: 0
  Maximum write same length: 0x200 blocks
```

Maximum transfer length value is set to 512 blocks. This is no accident – QEMU read the host max_sectors_kb and found it to be 256 kilobytes. One block is 512 bytes, so 512 blocks equals 256 kilobytes. This means that, from the guest point of view, the SCSI device is reporting a maximum capability that matches the max_sectors_kb setting it has on the host.

And this allows the guest to set up the max_sectors_kb value of the pass-through device:

```code
# cat /sys/block/sdb/queue/max_sectors_kb
256
```

### virtio-scsi

The virtio-scsi back end allows the guest to directly send SCSI requests back to the real device. Its functioning is shown in the following figure.

![Figure3](/assets/images/qemu_SCSI_VPD3.png)
{: style="text-align: center"}

All SCSI commands responses are sent by the real device, passing through QEMU. This mechanism allows the guest device to use all the features that the real device implements. Read and write requests from the guest are also sent directly to the real device.

Using the same environment from the virtio-blk example, the QEMU command line to use virtio-scsi is similar, but scsi-hd is changed to scsi-block”:

```code
sudo qemu-system-ppc64 (...)
-device virtio-scsi-pci,id=scsi0,bus=pci.0,addr=0x2 \
-device scsi-block,bus=scsi0.0,channel=0,scsi-id=0,lun=2,drive=drive-scsi0-0-0-2,id=scsi0-0-0-2 \
-drive file=/dev/disk/by-id/scsi-3600605b000a2c110ff0004053d84a61b,format=raw,if=none,id=drive-scsi0-0-0-2,cache=none
```

Using Libvirt, comparing with the virtio-blk example, change device=’disk’ to device=’lun’ to use virtio-scsi:

```xml
<disk type='block' device='lun'>
    <driver name='qemu' type='raw' cache='none'/>
    <source dev='/dev/disk/by-id/scsi-3600605b000a2c110ff0004053d84a61b'/>
    <target dev='sdc' bus='scsi'/>
    <address type='drive' controller='0' bus='0' target='0' unit='2'/>
</disk>
```


Inside the guest, issuing an Inquiry request to /dev/sdb using sg_inq gives us the information about the real device:

```code
# sg_inq /dev/sdb
standard INQUIRY:
  PQual=0  Device_type=0  RMB=0  version=0x05  [SPC-3]
  [AERC=0]  [TrmTsk=0]  NormACA=0  HiSUP=0  Resp_data_format=2
  SCCS=0  ACC=0  TPGS=0  3PC=0  Protect=0  [BQue=0]
  EncServ=0  MultiP=0  [MChngr=0]  [ACKREQQ=0]  Addr16=0
  [RelAdr=0]  WBus16=0  Sync=0  Linked=0  [TranDis=0]  CmdQue=1
  [SPI: Clocking=0x0  QAS=0  IUS=0]
    length=96 (0x60)   Peripheral device type: disk
 Vendor identification: LSI
 Product identification: MR9361-8i
 Product revision level: 4.23
 Unit serial number: 001ba6843d050400ff10c1a200b00506
```

The available VPD pages of the virtual device matches the pages that the real hardware supports:

```code
# sg_vpd /dev/sdb
Supported VPD pages VPD page:
  Supported VPD pages [sv]
  Unit serial number [sn]
  Device identification [di]
```

Aside from these differences, QEMU does the same setup strategy with the max_sectors_kb parameter described in “max_sectors_kb and Block Limits VPD” section when using virtio-scsi: QEMU intercepts the Block Limits VPD response from the real device that is addressed to the guest, changes the Maximum Transfer Length field, and then forwards it to the guest. Note that, in this case, this mechanism is bounded to the support of the Block Limits VPD page by the SCSI device in the host (which brings us to the problem that we want to discuss).



### The Block Limits problem with virtio-scsi
The Block Limits VPD page is optional – although many SCSI devices chooses to support it, its absence doesn’t hurt the SCSI specification.

This has a direct impact in a QEMU guest that uses SCSI pass-through with virtio-scsi. If the SCSI device does not support it, there will be no Block Limits VPD message between the guest and the SCSI device. Without this message, there is no way to let the guest know of the max_sectors_kb setting of the host. This means that the guest will take a default value for the parameter, which can be incompatible with the host side parameter, causing the guest device to malfunction.

Using the setup from “virtio-scsi” section, we can see that there is no Block Limits support for the virtio-scsi device of the guest:

```code
# sg_vpd /dev/sdb
Supported VPD pages VPD page:
  Supported VPD pages [sv]
  Unit serial number [sn]
  Device identification [di]
```

This shows that there is no Block Limits support, meaning that the guest configured max_sectors_kb with a default value, as given below:

```code
# cat /sys/block/sdb/queue/max_sectors_kb
1280
```

In this case, it is a value greater than the one in the host side:

```code
$ cat /sys/block/sdb/queue/max_sectors_kb
256
```

What will happen is that the guest SCSI device will send requests bigger than what the host can handle, that is, it can’t be used to read or write.

Performing a read test with dd:

```code
# dd if=/dev/sdb of=/dev/null
[   48.177632] sd 0:0:0:2: [sdb] tag#0 FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE
[   48.177831] sd 0:0:0:2: [sdb] tag#0 Sense Key : Aborted Command [current]
[   48.177933] sd 0:0:0:2: [sdb] tag#0 Add. Sense: I/O process terminated
[   48.178035] sd 0:0:0:2: [sdb] tag#0 CDB: Read(10) 28 00 00 00 02 00 00 04 00 00
[   48.178157] print_req_error: I/O error, dev sdb, sector 512
[   48.178413] sd 0:0:0:2: [sdb] tag#0 FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE
[   48.178621] sd 0:0:0:2: [sdb] tag#0 Sense Key : Aborted Command [current]
[   48.178722] sd 0:0:0:2: [sdb] tag#0 Add. Sense: I/O process terminated
[   48.178821] sd 0:0:0:2: [sdb] tag#0 CDB: Read(10) 28 00 00 00 06 00 00 08 00 00
[   48.178935] print_req_error: I/O error, dev sdb, sector 1536
(...)
```

Performing a write test with dd:

```code
# dd if=/dev/zero of=/dev/sdb
[  172.361520] scsi_io_completion: 30 callbacks suppressed
[  172.368064] sd 0:0:0:2: [sdb] tag#67 FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_SENSE
[  172.368174] sd 0:0:0:2: [sdb] tag#67 Sense Key : Aborted Command [current]
[  172.368266] sd 0:0:0:2: [sdb] tag#67 Add. Sense: I/O process terminated
[  172.368358] sd 0:0:0:2: [sdb] tag#67 CDB: Write(10) 2a 00 00 00 00 00 00 0a 00 00
[  172.368464] print_req_error: 30 callbacks suppressed
[  172.368539] print_req_error: I/O error, dev sdb, sector 0
[  172.368613] Buffer I/O error on dev sdb, logical block 0, lost async page write
(...)
```

If the guest was able to boot up to the prompt, there is a way to work around this issue.


## Workaround
To work around the SCSI sense error, set the max_sectors_kb parameter in the guest operating system to match the value that the device has on the host operating system. You can perform one of the following:

Run the echo command Set the value in the /sys/block/ directory. If the max_sectors_kb parameter in the host operating system is 256, set it to the same value in the guest operating system.

```code
`$` echo 256 > /sys/block/
```

This process can be automated to persist guest restart. An alternative method is to add the echo command in the /etc/rc.local file in the guest operating system. In this example, this is done for a /dev/sda SCSI device in the guest operating system.

```code
# cat /etc/rc.local
(…)
echo 256 > /sys/block/sda/queue/max_sectors_kb
```

You can also use udev rules to achieve this. However, adding the echo command in the /etc/rc.local file is an alternative method.

Set the value in libvirt You can set the value of the max_sectors_kb parameter directly in the libvirt XML file, forcing the whole SCSI bus to not surpass the value you want:

```xml
<controller type='scsi' index='0' model='virtio-scsi'>
<driver max_sectors='512'/>  <--"512" if you want to set max_sectors_kb to 256 -->
<address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
</controller>
```

This impacts all the SCSI devices that use this controller. You can use this approach when you want to install a new operating system that uses a SCSI pass-through disk that is affected by this issue.

To echo the right value to the /sys/block/ directory during a guest install operation, you must access a system terminal during the installation process and change the value of the max_sectors_kb parameter before the installation starts to write in the disk. Hence, you can set the value in libvirt. If the guest operating system is already installed, the approach described in the first method is less restrictive because it does not affect other devices.

Note that there will be times (a guest installation uses the virtio-scsi device and there is no way to set the parameter beforehand** that even these workarounds won’t suffice. In this case, the user would need to either remove the virtio-scsi disks or use virtio-blk instead during the install process.


## A solution using emulation
The workarounds for this virtio-scsi problem have the following drawbacks that can’t be easily ignored:

* It can be troublesome if the guest has many pass-through devices that needs this tuning
* If a change in max_sectors_kb is made in the host side, manual change in the guests will also be required.
* During an OS installation, it is difficult (and sometimes not possible) to go to a terminal and change the max_sectors_kb parameter prior to the installation.

A better way would be to fix this situation from inside QEMU. The author proposed a fix that relies on the already available emulation from virtio-blk and adjustments in the virtio-scsi back end, allowing the guest to query for the Block Limits page even when the SCSI hardware doesn’t support it. we’ll go through the concepts of the developed solution now.

### Guest must always ask for the Block Limits VPD page
To fix the max_sectors_kb issue with virtio-scsi, using the existing mechanism described in “max_sectors_kb and Block Limits VPD” section, the guest must always query for the Block Limits page regardless of the hardware supporting it or not. There is no way to make the guest aware of the proper setting otherwise.

However, all SCSI messages are proxy to the real SCSI hardware, which isn’t aware of what QEMU wants to accomplish. In a reply to an Inquiry message fetching the available pages, the real device will only advertise the Block Limits page if it supports it.

As seen in “SCSI Inquiry command” section, to query all available pages, an Inquiry message with the EVPD bit set is sent to the device. The format of the reply to this Inquiry request is shown in Figure 4.

![Figure4](/assets/images/qemu_SCSI_VPD4.png)
{: style="text-align: center"}


This is a variable size message where byte 3 is the length of the supported page list, which starts at byte 4. Considering our last example:

```code
# sg_vpd /dev/sdb
Supported VPD pages VPD page:
  Supported VPD pages [sv]
  Unit serial number [sn]
  Device identification [di]
```

Refer to the following hexadecimal format of the response:

```code
byte 0: 00
byte 1: 00
byte 2: 00
byte 3: 03
byte 4 and above: 00 80 83
```

Byte 3 indicates that the length of the page list is 3 bytes. Byte 4 up to 6 contains the list, which is 00 (supported VPD pages), 80 (unit serial number) and 83 (device identification).

This response passes through QEMU untouched and the guest will never ask for the Block Limits page. But we want the guest to ask for the Block Limits page even if the hardware doesn’t support it.

But, because we know how the guest will interpret it, we are able to change the response before it is delivered, adding the Block Limits in the page list. For each Inquiry with EVPD set response QEMU receives, check if the page b0h (Block Limits) is in the page list that is returned. If it is present, there is nothing to be done. Otherwise, we’ll add *b0 at the end of the page list and increment the page length information (byte 3).

* In this case the problem doesn’t occur – the hardware has Block Limits support and everything work as described in “max_sectors_kb and Block Limits VPD” section.

Refer to the following C code snippet that represents the idea. Considering that buf is a byte buffer with a SCSI message response:

```C
if (buf[1] == 0x00) { /* Buffer is a Supported VPD pages reply */
    uint8_t page_len = buf[3]; /* page length */

    /* See if Block Limits page 0xb0 is supported by the hardware.
     * Note that page_len does not consider the previous 3 bytes,
     * thus page_len + 4 is the total size of the buffer.
     */
    for (int i = 4; i < page_len + 4; i++) {
        if (buf[i] == 0xb0) {
            /* Nothing to be done*/
            return;
        }
    }
    /* Add 0xb0 at the end of the page list. Increment page_len */
    buf[page_len + 4] = 0xb0;
    buf[3] = ++page_len;
}
```

![Figure5](/assets/images/qemu_SCSI_VPD5.png)
{: style="text-align: center"}

Doing this change, the guest will be aware of Block Limits support, but the max_sectors_kb value is still wrong if the SCSI device does not support it. The guest will send Block Limits requests to the device and will get an error. We can see this behavior by fetching the available VPD pages and trying to get the Block Limits information inside the guest.

```code
# sg_vpd /dev/sdb
Supported VPD pages VPD page:
  Supported VPD pages [sv]
  Unit serial number [sn]
  Device identification [di]
  Block limits (SBC) [bl]

# sg_vpd --page=bl /dev/sdb
VPD page=0xb0
fetching VPD page failed
```

This is expected and will be handled by emulating the Block Limits VPD response.

### Emulate the Block Limits response if necessary

In section “virtio-blk“, we saw that the virtio-blk back end emulates all the SCSI replies that are sent back to the guest kernel. We also verified that QEMU implements the Block Limits VPD page in this case. This means that we already have code inside QEMU that can be used to solve the problem in virtio-scsi. If the guest sends a Block Limits response and an error is returned from the hardware, we can deliver an emulated Block Limits reply from the virtio-blk code, which has the max_sectors_kb parameter already considered, and the guest can properly set up the device.

![Figure6](/assets/images/qemu_SCSI_VPD6.png)
{: style="text-align: center"}


Note that this will only happen if the guest knows about the Block Limits support, meaning that we’ll need to make sure that we advertise it all the time using the code we discussed earlier.

The final version of the fix that was accepted in QEMU was enhanced on top of that. Instead of checking every Inquiry EVPD message, QEMU will fire an Inquiry supported VPD page request to the device right after the virtual machine starts. If the SCSI device does not advertise Block Limits support, an internal scsi-block flag called needs_vpd_emulation is set. This flag is then checked every time an Inquiry reply or a SCSI error comes from the hardware to QEMU to see if this is a case of either changing the Inquiry reply or emulating the Block Limits page.

A QEMU guest may have several scsi-block devices at the same time, and this flag allows a single verification at machine start for each scsi-block device instead of doing it for every Inquiry response or SCSI error. (Refer VPD Block Limits emulation implementation for more details.) Figure 7 illustrates all messages and events related to the fix that is available publicly in QEMU.


![Figure7](/assets/images/qemu_SCSI_VPD7.png)
{: style="text-align: center"}


## Wrapping up

The max_sectors_kb issue found and fixed in QEMU is an example of how flexible and robust virtualization technologies must be to support a great array of hardware, aiming to provide the best service available to customers.

The idea of using the Maximum Transfer Length field to insert the max_sectors_kb value of the Linux host andallowing the guest to properly set up the SCSI device is ingenious. But, this couldn’t fix all cases because it was reliant of hardware support for an optional VPD page, something that we can’t take for granted.

The work reported in this article, covering this corner case, makes the QEMU virtio-scsi layer more robust and convenient for users of SCSI pass-through devices.


{% include disqus-comments.html %}
