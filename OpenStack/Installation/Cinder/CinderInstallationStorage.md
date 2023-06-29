Perform these steps on the storage node.

Here is the two disks on my storage node: /vda/ and /vdb/


## Prerequisites

We need to perform a basic setup of LVM on each storage node, i.e. we need to create physical volumes and a volume group.

- Install the supporting utility packages:

```
    apt install lvm2 thin-provisioning-tools
```

- Create the LVM volume group cinder-volumes:

```
    vgcreate cinder-volumes /dev/vdb
```

- Only instances can access Block Storage volumes. However, the underlying operating system manages the devices associated with the volumes. By default, the LVM volume scanning tool scans the /dev directory for block storage devices that contain volumes. If projects use LVM on their volumes, the scanning tool detects these volumes and attempts to cache them which can cause a variety of problems with both the underlying operating system and project volumes. You must reconfigure LVM to scan only the devices that contain the cinder-volumes volume group. Edit the /etc/lvm/lvm.conf file and complete the following actions:

+ In the devices section, add a filter that accepts the /dev/sdb device and rejects all other devices:

```
    devices {
    ...
    filter = [ "a/vdb/", "r/.*/"]
```

    Each item in the filter array begins with a for accept or r for reject and includes a regular expression for the device name. The array must end with r/.*/ to reject any remaining devices. You can use the vgs -vvvv command to test filters.


## Install and configure components

- Install the packages:

```
    apt install cinder-volume
```

- Edit the /etc/cinder/cinder.conf file and complete the following actions:

```
    [database]
    # ...
    connection = mysql+pymysql://cinder:CINDER_DBPASS@ngoc-ctl/cinder

    [DEFAULT]
    # ...
    transport_url = rabbit://openstack:ngoc@ngoc-ctl
    enabled_backends = lvm
    glance_api_servers = http://ngoc-ctl:9292
    my_ip = 192.168.30.177
    auth_strategy = keystone


    [keystone_authtoken]
    # ...
    www_authenticate_uri = http://ngoc-ctl:5000
    auth_url = http://ngoc-ctl:5000
    memcached_servers = ngoc-ctl:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = cinder
    password = CINDER_PASS


    [lvm]
    # ...
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volume_group = cinder-volumes
    target_protocol = iscsi
    target_helper = tgtadm

    [oslo_concurrency]
    # ...
    lock_path = /var/lib/cinder/tmp
```

## Finalize installation

- Restart the Block Storage volume service including its dependencies:

```
    service tgt restart
    service cinder-volume restart
```

## Verify operation 

- Source the admin credentials to gain access to admin-only CLI commands: 

```
    . admin-openrc
```

- List service components to verify successful launch of each process 

```
    openstack volume service list
```
