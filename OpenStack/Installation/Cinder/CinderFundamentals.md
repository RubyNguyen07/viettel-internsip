## Cinder 

The Block Storage service (cinder) provides block storage devices to guest instances. The method in which the storage is provisioned and consumed is determined by the Block Storage driver, or drivers in the case of a multi-backend configuration. There are a variety of drivers that are available: NAS/SAN, NFS, iSCSI, Ceph, and more.

The Block Storage API and scheduler services typically run on the controller nodes. Depending upon the drivers used, the volume service can run on controller nodes, compute nodes, or standalone storage nodes.

The OpenStack Block Storage service (Cinder) adds persistent storage to a virtual machine. Block Storage provides an infrastructure for managing volumes, and interacts with OpenStack Compute to provide volumes for instances. The service also enables management of volume snapshots, and volume types.

The Block Storage service consists of the following components:

- cinder-api: Accepts API requests, and routes them to the cinder-volume for action.

- cinder-volume: Interacts directly with the Block Storage service, and processes such as the cinder-scheduler. It also interacts with these processes through a message queue. The cinder-volume service responds to read and write requests sent to the Block Storage service to maintain state. It can interact with a variety of storage providers through a driver architecture.

- cinder-scheduler daemon: Selects the optimal storage provider node on which to create the volume. A similar component to the nova-scheduler.

- cinder-backup daemon: The cinder-backup service provides backing up volumes of any type to a backup storage provider. Like the cinder-volume service, it can interact with a variety of storage providers through a driver architecture.

- Messaging queue
Routes information between the Block Storage processes.

### Logical Volume Manager (LVM)

The default volume back end uses local volumes managed by LVM. This driver supports different transport protocols to attach volumes, currently iSCSI and iSER.

The set of operating system commands, library subroutines, and other tools that allow you to establish and control logical volume storage is called the Logical Volume Manager (LVM).

The LVM controls disk resources by mapping data between a more simple and flexible logical view of storage space and the actual physical disks. The LVM does this using a layer of device-driver code that runs above traditional disk device drivers.

The LVM consists of the logical volume device driver (LVDD) and the LVM subroutine interface library. The logical volume device driver (LVDD) is a pseudo-device driver that manages and processes all I/O. It translates logical addresses into physical addresses and sends I/O requests to specific device drivers. The LVM subroutine interface library contains routines that are used by the system management commands to perform system management tasks for the logical and physical volumes of a system.

### Storage networks 

In the early days of computing, when persistent mass storage was introduced, storage devices where typically directly attached to a server, similar to the hard disk in your PC or laptop computer which is sitting in same enclosure as your motherboard and directly connected to it. In order to communicate with such a storage device, there would usually be some sort of controller on the motherboard which would use some low-level protocol to talk to a controller on the storage device.

A protocol to achieve this which is (still) very popular in the world of Intel PCs is the SATA protocol, but this is by far not the only one. In most enterprise storage solutions, another protocol called SCSI (small computer system interface) is still dominating, which was originally also used in the consumer market by companies like Apple. Let us quickly summarize some terms that are relevant when dealing with SCSI based devices.

First, every device on a SCSI bus has a SCSI ID. As a typical SCSI storage device may expose more than one disk, these disks are represented by logical unit numbers (LUNs). Generally speaking, every object that can receive a SCSI command is a logical unit (there are also logical units that do not represent actuals disks, but controllers). Each SCSI device can encompass more than one LUN. A SCSI device could, for instance, be a RAID array that exposes two logical disks, and each of these disks would then be addressable as a separate LUN.

When devices communicate over the SCSI bus, one of them acts as initiator and one of them acts as target. If, for instance, a host controller wants to read data from a SCSI hard disk, the host controller is the initiator, and the controller of the hard disk is the target. The initiator can send commands like “read a block” to the target, and the target will reply with data and / or a status code.

Now imagine a data center in which there is a large number of servers, each of which being equipped with a direct attached storage device. The servers might be connected by a network, but each disk (or other storage device like tape or a removable media drive) is only connected to one server. This setup is simple, but has a couple of drawbacks. First, if there is some space available on a disk, it cannot easily be made available for other servers, so that the overall utilization is low. Second, topics like availability, redundancy, backups, proper cooling and so forth have to be done individually for each server. And, last but not least, physical maintenance can be difficult if the servers are distributed over several locations.

![Old architecture](https://leftasexercise.files.wordpress.com/2019/12/dasd-e1577533102916.png)

For those reasons, an alternative architecture has evolved over time, in which storage capacity is centralized. Instead of having one disk attached to each server, most of the storage capacity is moved into a central storage appliance. This appliance is then connected to each server via a (typically dedicated) network, hence the term SAN – storage attached network that describes this sort of architecture (often, each server would still have a small disk as a primary partition for the operating system and booting, but not even this is actually required).

![Newer architecture](https://leftasexercise.files.wordpress.com/2019/12/san-1-e1577533514932.png)

Finally, there is a third possible architecture, which is becoming increasingly popular in the context of cloud and container platforms – distributed storage systems. With this approach, storage is still separated from the compute capacity and connected using a network, but instead of having a small number of large storage appliances that pool the available storage capacity, these solutions take a comparatively large number of smaller nodes, often commodity hardware, which distribute and replicate data to form a large, highly available virtual storage system. Examples for this type of solutions are the HDFS file system used by Hadoop, Ceph or GlusterFS.

![Newer architecture](https://leftasexercise.files.wordpress.com/2019/12/distributedstorage.png)




### iSCSI

Internet Small Computer Systems Interface or iSCSI is a storage area networking (SAN) protocol. It is an Internet Protocol-based networking standard for transferring data carrying SCSI commands over a TCP/IP network. iSCSI links data storage facilities and provides block-level access to storage devices. (used to build storage networks utilizing SCSI capable devices based on an underlying IP network)

iSCSI allows two hosts to interpose and exchange SCSI commands by using IP networks that take a high-performance local storage bus, emulate it over a network, and create a storage area network (SAN). The protocol encapsulates SCSI commands, assembles data in TCP/IP layer packets, and sends them using a point-to-point connection.

iSCSI performs by transporting block-level data between an iSCSI initiator and an iSCSI target, the iSCSI initiator placed on a server, and the iSCSI target placed on a storage device. After the packet arrives at the iSCSI target, the protocol disassembles the packets and separates the SCSI commands so the storage can be visible to the operating system (OS).

iSCSI can run over existing IP infrastructure and does not require any dedicated cabling; as a result, iSCSI is often seen as low-cost than its alternative, such as Fibre Channel. The iSCSI protocol can communicate with arbitrary types of SCSI devices, and system administrators widely use the protocol to allow servers to access disk volumes on storage arrays. But the performance can be degraded if iSCSI is not operated on a dedicated network or subnet.



### Keywords 
- LVM driver 
- iSCSI transport
- SCSI ("scuzzy"): SCSI is a once-popular type of connection for storage and other devices in a PC. The term refers to the cables and ports used to connect certain types of hard drives, optical drives, scanners, and other peripheral devices to a computer.
- RAID (redundant array of independent disks): a type of storage that writes data across multiple drives within the same system. 
- SCSI bus: the interface is to provide host computer systems with connections to a variety of peripheral devices, including disk subsystems, tape subsystems, printers, scanners, optical devices, communication devices, and libraries.
- thin-provisioning-tools: A suite of tools for manipulating the metadata of the dm-thin, dm-cache and dm-era device-mapper targets.
- volume group: 

![demo image](https://www.redhat.com/sysadmin/sites/default/files/styles/embed_small/public/2020-03/basic-lvm-volume_0.png?itok=yiz5yMKN)



### Resources
https://docs.openstack.org/cinder/victoria/install/
https://www.enterprisestorageforum.com/hardware/what-is-iscsi-and-how-does-it-work/
https://docs.openstack.org/cinder/rocky/configuration/block-storage/drivers/lvm-volume-driver.html
https://www.ibm.com/docs/en/aix/7.2?topic=management-logical-volume-manager
https://leftasexercise.com/2020/04/06/openstack-cinder-foundations-storage-networks-iscsi-luns-and-all-that/
https://www.redhat.com/sysadmin/create-volume-group 