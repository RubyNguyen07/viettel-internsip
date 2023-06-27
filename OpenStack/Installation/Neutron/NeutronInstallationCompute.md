## Install and configure compute node

For compute node, I have to configure the same networking option I chose for the controller node. 
### Install the components 

```
    apt install neutron-linuxbridge-agent
```

### Configure the common component

The Networking common component configuration includes the authentication mechanism, message queue, and plug-in.

- Edit the /etc/neutron/neutron.conf file and complete the following actions.

- In the [database] section, comment out any connection options because compute nodes do not directly access the database.

```
    [DEFAULT]
    # ...
    transport_url = rabbit://openstack:RABBIT_PASS@ngoc-ctl
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
    username = neutron
    password = NEUTRON_PASS

    [oslo_concurrency]
    # ...
    lock_path = /var/lib/neutron/tmp
```

### Configure the Linux bridge agent

The Linux bridge agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups.

- Edit the /etc/neutron/plugins/ml2/linuxbridge_agent.ini file and complete the following actions:

```
# In the [linux_bridge] section, map the provider virtual network to the provider physical network interface:

[linux_bridge]
physical_interface_mappings = provider:eth0


[vxlan]
enable_vxlan = true
local_ip = 192.168.30.143
l2_population = true


[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

- Ensure your Linux operating system kernel supports network bridge filters by verifying all the following sysctl values are set to 1: 

```    
    systemctl net.bridge.bridge-nf-call-iptables

    systemctl net.bridge.bridge-nf-call-ip6tables
```

### Configure the Compute service to use the Networking service

- Edit the /etc/nova/nova.conf file and complete the following actions:

```
[neutron]
# ...
auth_url = http://ngoc-ctl:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```

### Finalize installation

- Restart the Compute service:

```
    service nova-compute restart
```

- Restart the Linux bridge agent:

```
    service neutron-linuxbridge-agent restart
```

### Verify operation 

- List agents to verify successful launch of the neutron agents:

``
    openstack network agent list
``

The output should indicate four agents on the controller node and one agent on each compute node.