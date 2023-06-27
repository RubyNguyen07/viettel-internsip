## Install and configure controller node

There are two networking options: Provider (option 1) and Self-service (option 2). In this guide, I will deploy the service using the self-service architecture. 

Option 1 deploys the simplest possible architecture that only supports attaching instances to provider (external) networks. No self-service (private) networks, routers, or floating IP addresses. Only the admin or other privileged user can manage provider networks.

Option 2 augments option 1 with layer-3 services that support attaching instances to self-service networks. The demo or other unprivileged user can manage self-service networks including routers that provide connectivity between self-service and provider networks. Additionally, floating IP addresses provide connectivity to instances using self-service networks from external networks such as the Internet.

### Prerequisites

- Use the database access client to connect to the database server as the root user:

```
    mysql -u root -p
```

- Create the neutron database:

```
MariaDB [(none)] CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'NEUTRON_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'NEUTRON_DBPASS';
```

- Exit the database access client.

- Source the admin credentials to gain access to admin-only CLI commands:

```
    . admin-openrc
```

- Create the neutron user:

```
    openstack user create --domain default --password-prompt neutron
```

- Add the admin role to the neutron user:

```
openstack role add --project service --user neutron admin
```

- Create the neutron service entity:

```
    openstack service create --name neutron --description "OpenStack Networking" network
```

- Create the Networking service API endpoints:

```
    openstack endpoint create --region RegionOne network public http://ngoc-ctl:9696
    openstack endpoint create --region RegionOne network internal http://ngoc-ctl:9696
    openstack endpoint create --region RegionOne network admin http://ngoc-ctl:9696
```

### Install the components 

```
apt install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

### Configure the server component 

- Edit the /etc/neutron/neutron.conf file and complete the following actions:

```
    [database]
    # ...
    connection = mysql+pymysql://neutron:NEUTRON_DBPASS@ngoc-ctl/neutron

    [DEFAULT]
    # ...
    core_plugin = ml2
    service_plugins = router # there are many services allowed e.g. firewall, vpn, ...
    allow_overlapping_ips = true 
    transport_url = rabbit://openstack:RABBIT_PASS@ngoc-ctl
    auth_strategy = keystone
    notify_nova_on_port_status_changes = true
    notify_nova_on_port_data_changes = true


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


    [nova]
    # ...
    auth_url = http://ngoc-ctl:5000
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    region_name = RegionOne
    project_name = service
    username = nova
    password = NOVA_PASS

    [oslo_concurrency]
    # ...
    lock_path = /var/lib/neutron/tmp
```

### Configure the Modular Layer 2 (ML2) plug-in

The ML2 plug-in uses the Linux bridge mechanism to build layer-2 (bridging and switching) virtual networking infrastructure for instances.

- Edit the /etc/neutron/plugins/ml2/ml2_conf.ini file and complete the following actions:

```
    [ml2]
    # ...
    type_drivers = flat,vlan,vxlan
    tenant_network_types = vxlan
    mechanism_drivers = linuxbridge,l2population
    extension_drivers = port_security 
    # Neutron’s security group always applies anti-spoof rules on the VMs. This allows traffic to originate and terminate at the VM as expected, but prevents traffic to pass through the VM. 

    [ml2_type_flat]
    # ...
    flat_networks = provider

    # In the [ml2_type_vxlan] section, configure the VXLAN network identifier range for self-service networks:
    [ml2_type_vxlan]
    # ...
    vni_ranges = 1:1000

    [securitygroup]
    # ...
    enable_ipset = true
```

! Note:  The Linux bridge agent only supports VXLAN overlay networks.


### Configure the Linux bridge agent

- Edit the /etc/neutron/plugins/ml2/linuxbridge_agent.ini file and complete the following actions:

```
    [linux_bridge]
    physical_interface_mappings = provider:eth0 # Map provider virtual network to the physical one 

    [vxlan]
    enable_vxlan = true
    local_ip = 192.168.30.39
    l2_population = true

    [securitygroup]
    # ...
    enable_security_group = true
    firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

- Ensure your Linux operating system kernel supports network bridge filters by verifying all the following sysctl values are set to 1: net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables

### Configure the layer-3 agent
The Layer-3 (L3) agent provides routing and NAT services for self-service virtual networks.

- Edit the /etc/neutron/l3_agent.ini file and complete the following actions:

```
    [DEFAULT]
    # ...
    interface_driver = linuxbridge
```

### Configure the DHCP agent
The DHCP agent provides DHCP services for virtual networks.

- Edit the /etc/neutron/dhcp_agent.ini file. In the [DEFAULT] section, configure the Linux bridge interface driver, Dnsmasq DHCP driver, and enable isolated metadata so instances on provider networks can access metadata over the network:

```
    [DEFAULT]
    # ...
    interface_driver = linuxbridge
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    enable_isolated_metadata = true
```

### Configure the metadata agent

The metadata agent provides configuration information such as credentials to instances.

- Edit the /etc/neutron/metadata_agent.ini file and complete the following actions:

```
    [DEFAULT]
    # ...
    nova_metadata_host = ngoc-ctl
    metadata_proxy_shared_secret = METADATA_SECRET
```

### Configure the Compute service to use the Networking service

- Edit the /etc/nova/nova.conf file and perform the following actions:


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
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET


### Finalize installation

- Populate the database:

```
    su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
  ```

- Restart the Compute API service:

```
    service nova-api restart
```

- Restart the Networking services. For both networking options:

```
    service neutron-server restart
    service neutron-linuxbridge-agent restart
    service neutron-dhcp-agent restart
    service neutron-metadata-agent restart
```

- For networking option 2, also restart the layer-3 service:

```
    service neutron-l3-agent restart
```