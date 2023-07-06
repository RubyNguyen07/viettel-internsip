## Launch and manage instances

There are several different kinds of sources to boot from, but they all need to create some sort of virtual disk for the virtual machine to boot. The virtual disk can use either ephemeral storage or volume block storage. When launching an instance, you have several Boot Source options:

Image: Launches an instance from the image you choose onto either an ephemeral disk or a new volume disk.
Instance Snapshot: Launches an instance from the instance snapshot you choose onto either an ephemeral disk or a new volume disk.
Volume: Launches an instance from an existing bootable volume.
Volume Snapshot: Creates a volume from the volume snapshot you choose and then launches an instance using that new bootable volume.

Note: When you launch instance on Horizon, can choose to create a new volume at the same time. Later on, you can boot new instances from that volume (but have to delete that instance running on that volume).  

### Ephemeral vs Boot-From-Volume Instances 

In OpenStack, instances may be provisioned with either an ephemeral storage or persistent (volume) storage. 

Ephemeral Boot Disk:

- Ephemeral disks are created from an image and are tightly coupled with the virtual machine instance.
- The disk is created on the local storage of the compute node where the instance is running.
- Ephemeral disks are temporary in nature and are typically used for non-persistent data or workloads that do not require data persistence beyond the lifespan of the instance.
- Any data written to the ephemeral disk is lost when the instance is terminated or stopped.
- Ephemeral disks are suitable for stateless workloads or situations where data persistence is not a requirement.

Volume Boot Disk:

- Volume boot disks are created from OpenStack Cinder volumes and provide persistent storage for virtual machine instances.
- The disk is created on a separate storage system (usually a SAN or NAS) and is attached to the instance at boot time.
- Volume boot disks allow data to persist even when the instance is terminated or stopped.
- You can detach the volume from one instance and attach it to another instance, allowing for data portability and migration.
- Volume boot disks are suitable for workloads that require data persistence, such as databases or applications that need to retain data across instance lifecycles.

### Manage an instance

You can resize or rebuild an instance. You can also choose to view the instance console log, edit instance or the security groups. Depending on the current state of the instance, you can pause, resume, suspend, soft or hard reboot, or terminate it.

## Resources 
https://docs.openstack.org/horizon/latest/user/launch-instances.html
https://help.dreamhost.com/hc/en-us/articles/217701757-What-s-the-difference-between-ephemeral-and-volume-boot-disks-
https://docs.openstack.org/nova/latest/user/launch-instance-from-volume.html