This section describes how to install and configure the Compute service on a compute node. 

## Install and configure components 

- Install the packages 

```
    apt install nova-compute
```

- Edit the /etc/nova/nova.conf file and complete the following actions:

```
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@ngoc-ctl
my_ip = 192.168.30.143

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://ngoc-ctl:5000/
auth_url = http://ngoc-ctl:5000/
memcached_servers = ngoc-ctl:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = NOVA_PASS
Replace NOVA_PASS with the password you chose for the nova user in the Identity service.


[vnc]
enabled = true
# The server component listens on all IP addresses 
server_listen = 0.0.0.0
# The address of server is set to the first interface IP address of the compute node 
server_proxyclient_address = 192.168.30.143
# Location where you can use a web browser to access remote consoles of instances on this compute node 
novncproxy_base_url = http://192.168.30.39:6080/vnc_auto.html

[glance]
# ...
api_servers = http://ngoc-ctl:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://ngoc-ctl:5000/v3
username = placement
password = PLACEMENT_PASS
```


## Finalize installation
- Determine whether your compute node supports hardware acceleration for virtual machines:

```
    egrep -c '(vmx|svm)' /proc/cpuinfo
```

If this command returns a value of one or greater, your compute node supports hardware acceleration which typically requires no additional configuration.

If this command returns a value of zero, your compute node does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM.

- Edit the [libvirt] section in the /etc/nova/nova-compute.conf file as follows:

```
    [libvirt]
    # ...
    virt_type = qemu
```

- Restart the Compute service:

```
    service nova-compute restart
```


! Note for the [vnc] section: 
- The server component listens on all IP addresses and the proxy component only listens on the management interface IP address of the compute node. The base URL indicates the location where you can use a web browser to access remote consoles of instances on this compute node.
- If the web browser to access remote consoles resides on a host that cannot resolve the controller hostname, you must replace controller with the management interface IP address of the controller node.


## Add the compute node to the cell database

Run the following commands on the controller node.

- Source the admin credentials to enable admin-only CLI commands, then confirm there are compute hosts in the database:

```
    . admin-openrc
    openstack compute service list --service nova-compute
    +----+-------+--------------+------+-------+---------+----------------------------+
    | ID | Host  | Binary       | Zone | State | Status  | Updated At                 |
    +----+-------+--------------+------+-------+---------+----------------------------+
    | 1  | node1 | nova-compute | nova | up    | enabled | 2017-04-14T15:30:44.000000 |
    +----+-------+--------------+------+-------+---------+----------------------------+
```

- Discover compute hosts:

```
    su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

## Verify operation 

Perform these commands on the controller node.

- Source the admin credentials to gain access to admin-only CLI commands:

```
    . admin-openrc
```
    
- List service components to verify successful launch and registration of each process:

    openstack compute service list

    +----+--------------------+------------+----------+---------+-------+----------------------------+
    | Id | Binary             | Host       | Zone     | Status  | State | Updated At                 |
    +----+--------------------+------------+----------+---------+-------+----------------------------+
    |  1 | nova-scheduler     | controller | internal | enabled | up    | 2016-02-09T23:11:15.000000 |
    |  2 | nova-conductor     | controller | internal | enabled | up    | 2016-02-09T23:11:16.000000 |
    |  3 | nova-compute       | compute1   | nova     | enabled | up    | 2016-02-09T23:11:20.000000 |
    +----+--------------------+------------+----------+---------+-------+----------------------------+

- List API endpoints in the Identity service to verify connectivity with the Identity service:

```
    openstack catalog list

    +-----------+-----------+-----------------------------------------+
    | Name      | Type      | Endpoints                               |
    +-----------+-----------+-----------------------------------------+
    | keystone  | identity  | RegionOne                               |
    |           |           |   public: http://controller:5000/v3/    |
    |           |           | RegionOne                               |
    |           |           |   internal: http://controller:5000/v3/  |
    |           |           | RegionOne                               |
    |           |           |   admin: http://controller:5000/v3/     |
    |           |           |                                         |
    | glance    | image     | RegionOne                               |
    |           |           |   admin: http://controller:9292         |
    |           |           | RegionOne                               |
    |           |           |   public: http://controller:9292        |
    |           |           | RegionOne                               |
    |           |           |   internal: http://controller:9292      |
    |           |           |                                         |
    | nova      | compute   | RegionOne                               |
    |           |           |   admin: http://controller:8774/v2.1    |
    |           |           | RegionOne                               |
    |           |           |   internal: http://controller:8774/v2.1 |
    |           |           | RegionOne                               |
    |           |           |   public: http://controller:8774/v2.1   |
    |           |           |                                         |
    | placement | placement | RegionOne                               |
    |           |           |   public: http://controller:8778        |
    |           |           | RegionOne                               |
    |           |           |   admin: http://controller:8778         |
    |           |           | RegionOne                               |
    |           |           |   internal: http://controller:8778      |
    |           |           |                                         |
    +-----------+-----------+-----------------------------------------+


- List images in the Image service to verify connectivity with the Image service:

```
    openstack image list

+--------------------------------------+-------------+-------------+
| ID                                   | Name        | Status      |
+--------------------------------------+-------------+-------------+
| 9a76d9f9-9620-4f2e-8c69-6c5691fae163 | cirros      | active      |
+--------------------------------------+-------------+-------------+
```

- Check the cells and placement API are working successfully and that other necessary prerequisites are in place:

```
    nova-status upgrade check

    +--------------------------------------------------------------------+
    | Upgrade Check Results                                              |
    +--------------------------------------------------------------------+
    | Check: Cells v2                                                    |
    | Result: Success                                                    |
    | Details: None                                                      |
    +--------------------------------------------------------------------+
    | Check: Placement API                                               |
    | Result: Success                                                    |
    | Details: None                                                      |
    +--------------------------------------------------------------------+
    | Check: Ironic Flavor Migration                                     |
    | Result: Success                                                    |
    | Details: None                                                      |
    +--------------------------------------------------------------------+
    | Check: Cinder API                                                  |
    | Result: Success                                                    |
    | Details: None                                                      |
    +--------------------------------------------------------------------+
    | Check: Policy Scope-based Defaults                                 |
    | Result: Success                                                    |
    | Details: None                                                      |
    +--------------------------------------------------------------------+
    | Check: Policy File JSON to YAML Migration                          |
    | Result: Success                                                    |
    | Details: None                                                      |
    +--------------------------------------------------------------------+
    | Check: Older than N-1 computes                                     |
    | Result: Success                                                    |
    | Details: None                                                      |
    +--------------------------------------------------------------------+
```