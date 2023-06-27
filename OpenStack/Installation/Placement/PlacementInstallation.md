This section describes how to install and configure the placement service when using Ubuntu packages on the controller node. 

## Prerequisites

- Use the database access client to connect to the database server as the root user:

```
    mysql
```
    
- Create the placement database and grant proper access to the database:

```
MariaDB [(none)]> CREATE DATABASE placement;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
  IDENTIFIED BY 'PLACEMENT_DBPASS';
```

- Exit the database access client.

## Configure User and Endpoints 

- Source the admin credentials to gain access to admin-only CLI commands:

```
    . admin-openrc
```

- Create a Placement service user using your chosen PLACEMENT_PASS:

```
    openstack user create --domain default --password-prompt placement
```

- Add the Placement user to the service project with the admin role:

```
    openstack role add --project service --user placement admin
```

- Create the Placement API entry in the service catalog:

```
    openstack service create --name placement --description "Placement API" placement
```

- Create the Placement API service endpoints:

```
    openstack endpoint create --region RegionOne placement public http://ngoc-ctl:8778
    openstack endpoint create --region RegionOne placement internal http://ngoc-ctl:8778
    openstack endpoint create --region RegionOne placement admin http://ngoc-ctl:8778
```

## Install and configure components 

- Install the packages 

```
    apt install placement-api
```

- Edit the /etc/placement/placement.conf file and complete the following actions:

```
[placement_database]
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@ngoc-ctl/placement

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://ngoc-ctl:5000/v3
memcached_servers = ngoc-ctl:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS
Replace PLACEMENT_PASS with the password you chose for the placement user in the Identity service.
```

- Populate the placement database:

```
su -s /bin/sh -c "placement-manage db sync" placement
```

## Finalize installation 

```
    service apache2 restart
```

## Verify operation 

```
    . admin-openrc
    placement-status upgrade check
    pip3 install osc-placement
    openstack --os-placement-api-version 1.2 resource class list --sort-column name
```

OR

```
. admin-openrc
sudo apt-get install jq
token=$(openstack token issue -f json | jq -r ".id") 
curl \
  -H "X-Auth-Token: $token" \
  -H "OpenStack-API-Version: placement 1.31"\
  "http://ngoc-ctl:8778/resource_classes" | jq
```
