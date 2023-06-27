Perform these steps on the storage node.


## Prerequisites

- Install the supporting utility packages:

```
    apt install lvm2 thin-provisioning-tools
```

- Create the LVM volume group cinder-volumes:

```
vgcreate cinder-volumes /dev/vdb
```


Only instances can access Block Storage volumes. However, the underlying operating system manages the devices associated with the volumes. By default, the LVM volume scanning tool scans the /dev directory for block storage devices that contain volumes. If projects use LVM on their volumes, the scanning tool detects these volumes and attempts to cache them which can cause a variety of problems with both the underlying operating system and project volumes. You must reconfigure LVM to scan only the devices that contain the cinder-volumes volume group. Edit the /etc/lvm/lvm.conf file and complete the following actions:

In the devices section, add a filter that accepts the /dev/sdb device and rejects all other devices:

devices {
...
filter = [ "a/sdb/", "r/.*/"]
Each item in the filter array begins with a for accept or r for reject and includes a regular expression for the device name. The array must end with r/.*/ to reject any remaining devices. You can use the vgs -vvvv command to test filters.