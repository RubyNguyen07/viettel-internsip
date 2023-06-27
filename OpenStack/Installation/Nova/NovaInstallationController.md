This section describes how to install and configure the Compute service, code-named nova, on the controller node.

## Prerequisites

- Use the database access client to connect to the database server as the root user:

```
    mysql

```

- Create the nova_api, nova, and nova_cell0 databases and grant proper access to the databases:

```
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
```

- Exit the database access client

- Source the admin credentials to gain access to admin-only CLI commands:

```
    . admin-openrc
```

- Create the Compute service credentials:

```
    openstack user create --domain default --password-prompt nova
    openstack role add --project service --user nova admin
    openstack service create --name nova --description "OpenStack Compute" compute
```

- Create the Compute API service endpoints:

```
    openstack endpoint create --region RegionOne compute public http://ngoc-ctl:8774/v2.1
    openstack endpoint create --region RegionOne compute internal http://ngoc-ctl:8774/v2.1
    openstack endpoint create --region RegionOne compute admin http://ngoc-ctl:8774/v2.1
```

- Install Placement service and configure user and endpoints. 


## Install and configure components 

- Install the packages:

```
    apt install nova-api nova-conductor nova-novncproxy nova-scheduler
```

- Edit the /etc/nova/nova.conf file and complete the following actions:

```
[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@ngoc-ctl/nova_api

[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@ngoc-ctl/nova

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@ngoc-ctl:5672/
my_ip = 192.168.30.39

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

[vnc]
enabled = true
# ...
# Server listens on the first interface IP address of the controller 
server_listen = $my_ip
# Proxy is set to be at the first interface IP address of the controller
server_proxyclient_address = $my_ip

[glance]
# ...
api_servers = http://ngoc-ctl:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp
#Due to a packaging bug, remove the log_dir option from the [DEFAULT] section.


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

- Populate the nova-api database:

```
    su -s /bin/sh -c "nova-manage api_db sync" nova
```

- Register the cell0 database:

```
    su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

- Create the cell1 cell:

```
    su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

- Populate the nova database:

```
    su -s /bin/sh -c "nova-manage db sync" nova
```

- Verify nova cell0 and cell1 are registered correctly:

```
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

## Finalize installation 

```
    service nova-api restart
    service nova-scheduler restart
    service nova-conductor restart
    service nova-novncproxy restart
```