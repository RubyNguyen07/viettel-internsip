## Cinder 

The Block Storage service (cinder) provides block storage devices to guest instances. The method in which the storage is provisioned and consumed is determined by the Block Storage driver, or drivers in the case of a multi-backend configuration. There are a variety of drivers that are available: NAS/SAN, NFS, iSCSI, Ceph, and more.

The Block Storage API and scheduler services typically run on the controller nodes. Depending upon the drivers used, the volume service can run on controller nodes, compute nodes, or standalone storage nodes.

The OpenStack Block Storage service (Cinder) adds persistent storage to a virtual machine. Block Storage provides an infrastructure for managing volumes, and interacts with OpenStack Compute to provide volumes for instances. The service also enables management of volume snapshots, and volume types.

The Block Storage service consists of the following components:

- cinder-api: Accepts API requests, and routes them to the cinder-volume for action.

- cinder-volume: Interacts directly with the Block Storage service, and processes such as the cinder-scheduler. It also interacts with these processes through a message queue. The cinder-volume service responds to read and write requests sent to the Block Storage service to maintain state. It can interact with a variety of storage providers through a driver architecture.

- cinder-scheduler daemon: Selects the optimal storage provider node on which to create the volume. A similar component to the nova-scheduler.

- cinder-backup daemon: The cinder-backup service provides backing up volumes of any type to a backup storage provider. Like the cinder-volume service, it can interact with a variety of storage providers through a driver architecture.

- Messaging queue: Routes information between the Block Storage processes.

### Logical Volume Manager (LVM)

The default volume back end uses local volumes managed by LVM. This driver supports different transport protocols to attach volumes, currently iSCSI and iSER.

The set of operating system commands, library subroutines, and other tools that allow you to establish and control logical volume storage is called the Logical Volume Manager (LVM).

The LVM controls disk resources by mapping data between a more simple and flexible logical view of storage space and the actual physical disks. The LVM does this using a layer of device-driver code that runs above traditional disk device drivers.

The LVM consists of the logical volume device driver (LVDD) and the LVM subroutine interface library. The logical volume device driver (LVDD) is a pseudo-device driver that manages and processes all I/O. It translates logical addresses into physical addresses and sends I/O requests to specific device drivers. The LVM subroutine interface library contains routines that are used by the system management commands to perform system management tasks for the logical and physical volumes of a system.

First, there are of course physical devices. These are ordinary block devices that the LVM will completely manage, or partitions on block devices. Technically, even though these devices are called physical devices in this context, these devices can themselves be virtual devices, which happens for instance if you run LVM on top of a software RAID. Logically,
the physical devices are divided further into physical extents. 

On the second layer, LVM now bundles one or several physical devices into a volume group. On top of that volume group, you can now create logical devices. These logical devices can be thought of as being divided into logical extents. LVM maps these logical extents to physical extents of the underlying volume group. Thus, a logical device is essentially a collection of physical extents of the underlying volume group which are presented to a user as a logical block device. On top of these logical volumes, you can then create file systems as usual.

![Components](https://leftasexercise.files.wordpress.com/2019/12/lvmterms.png)

Why would you want to do this? One obvious advantage is again based on the idea of pooling. A logical volume essentially pools the storage capacity of the underlying physical devices and LVM can dynamically assign space to logical devices. If a logical device starts to fill up while other logical devices are still mostly empty, an administrator can simply reallocate capacity between the logical devices without having to change the physical configuration of the system.

Another use case is virtualization. Given that there is sufficiently storage in your logical volume group, you can dynamically create new logical devices with a simple command, which can for instance be used to automatically provision volumes for cloud instances – this is how Cinder leverages the LVM as we will see later on.

Another useful functionality that LVM offers is a snapshot. When you create a snapshot, LVM will not simply create a physical copy. Instead, it will start to mark blocks which are changed after the snapshot has been taken as changed and only copy those blocks to a different location. This makes using the snapshot functionality very efficient.

#### LVM metadata daemon 

We have said above that LVM stores its state on the physical devices. This, however, is only a part of the story, as it would imply that whenever we use one of the tools introduced above, we have to scan all devices, which is slow and might interfere with other read or write access to the device.

For that reason, LVM comes with a metadata daemon, running as lvmetad in the background as a systemctl service. This daemon maintains a cache of the LVM metadata that a command like lvscan will typically use (you can see this if you try to run such a command as non-root, which will cause an error message while the tool is trying to connect to the daemon via a Unix domain socket).

The metadata daemon is also involved when devices are added (hotplug), removed, or changed. If, for example, a physical volume comes up, a Linux kernel mechanism known as udev informs LVM about this event, and when a volume group is complete, all logical volumes based on it are automatically activated (see the comment on use_lvmetad in the configuration file /etc/lvm/lvm.conf).

#### LVM snapshots

Let us now explore a very useful feature of LVM – efficiently creating COW (copy-on-write) snapshots.

When using copy-on-write, you would proceed differently. First, you would create a list of all extents. Then, you would start to monitor write activities on the original volume. As soon as an extent is about to be changed, you would mark it as changed and create a copy of that extent to preserve its content. For those extents, however, that have not yet changed since the snapshot has been created, you would not create a copy, but refer to the original content when someone tries to read from the snapshot, similar to a file system link.


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

![Diagram](https://leftasexercise.files.wordpress.com/2019/12/iscsientities.png)

### Architecture 

Essentially, Cinder consists of three main components which are running as independent processes and typically on different nodes.

First, there is the Cinder API server cinder-api. This is a WSGI-server running inside of Apache2. As the name suggests, cinder-api is responsible for accepting and processing REST API requests from users and other components of OpenStack and typically runs on a controller node.

Then, there is cinder-volume, the Cinder volume manager. This component is running on each node to which storage is attached (“storage node”) and is resonsible for managing this storage, i.e. preparing, maintaining and deleting virtual volumes and exporting these volumes so that they can be used by a compute node. And finally, Cinder comes with its own scheduler, which directs requests to create a virtual volume to an appropriate storage node.

![Architecture](https://leftasexercise.files.wordpress.com/2020/01/cinderarchitecture.png)

Cinder can use a variety of different storage backends, ranging from LVM managing local storage directly attached to a storage node over other standards like NFS and open source solutions like Ceph to vendor-specific drivers like Dell, Huawei, NetApp or Oracle – see the official support matrix for a full list. This is achieved by moving low-level access to the actual volumes into a volume driver. Similarly, Cinder uses various technologies to connect the virtual volumes to the compute node on which an instance that wants to use it is running, using a so-called target driver.

To better understand how Cinder operates, let us take a look at one specific use case – creating a volume and attaching to an instance. We will go in more detail on this and similar use cases in the next post, but what essentially happens is the following:

1. A user (for instance an administrator) sends a request to the Cinder API server to create a logical volume
2. The API server asks the scheduler to determine a storage node with sufficient capacity
3. The request is forwarded to the volume manager on the target node
4. The volume manager uses a volume driver to create a logical volume. With the default settings, this will be LVM, for which a volume group and underlying physical volumes have been created during installation. LVM will then create a new logical volume
5. Next, the administrator attaches the volume to an instance using another API request
6. Now, the target driver is invoked which sets up an iSCSI target on the storage node and a LUN pointing to the logical volume
7. The storage node informs the compute node about the parameters (portal IP and port, target name) that are needed to access this target
8. The Nova compute agent on the compute node invokes an iSCSI initiator which maps the iSCSI target into the local file system
9. Finally, the Nova compute agent asks the virtual machine driver (libvirt in our case) to attach this locally mapped device to the instance


### Keywords 
- LVM: The Logical Volume Manager (LVM) is a Linux-based system that provides an abstraction layer on top of physical disks to expose logical volumes to the operating system. The LVM back-end implements block storage as LVM logical partitions. On each host that will house block storage, an administrator must initially create a volume group dedicated to Block Storage volumes. Blocks are created from LVM logical volumes.
- iSCSI: Internet Small Computer Systems Interface (iSCSI) is a network protocol that operates on top of the Transport Control Protocol (TCP) for linking data storage devices. It transports data between an iSCSI initiator on a server and iSCSI target on a storage device.
- SCSI ("scuzzy"): SCSI is a once-popular type of connection for storage and other devices in a PC. The term refers to the cables and ports used to connect certain types of hard drives, optical drives, scanners, and other peripheral devices to a computer.
- RAID (redundant array of independent disks): a type of storage that writes data across multiple drives within the same system. 
- SCSI bus: the interface is to provide host computer systems with connections to a variety of peripheral devices, including disk subsystems, tape subsystems, printers, scanners, optical devices, communication devices, and libraries.
- thin-provisioning-tools: A suite of tools for manipulating the metadata of the dm-thin, dm-cache and dm-era device-mapper targets.
- volume group: 
- extents: An extent the smallest unit of storage that LVM manages.

![demo image](https://www.redhat.com/sysadmin/sites/default/files/styles/embed_small/public/2020-03/basic-lvm-volume_0.png?itok=yiz5yMKN)



### Resources
https://docs.openstack.org/cinder/victoria/install/
https://www.enterprisestorageforum.com/hardware/what-is-iscsi-and-how-does-it-work/
https://docs.openstack.org/cinder/rocky/configuration/block-storage/drivers/lvm-volume-driver.html
https://www.ibm.com/docs/en/aix/7.2?topic=management-logical-volume-manager
https://leftasexercise.com/2020/04/06/openstack-cinder-foundations-storage-networks-iscsi-luns-and-all-that/
https://www.redhat.com/sysadmin/create-volume-group 