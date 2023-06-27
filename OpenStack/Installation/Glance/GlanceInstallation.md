This section describes how to install and configure the Image service, code-named glance, on the controller node. For simplicity, this configuration stores images on the local file system.

## Prerequisites

- Use the database access client to connect to the database server as the root user:

```
    mysql
```

- Create the glance database and grant proper access to the glance database:

```
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
```

- Exit the database access client.

- Source the admin credentials to gain access to admin-only CLI commands:

```
    . admin-openrc
```

- Create the glance user

```
    openstack user create --domain default --password-prompt glance
```

- Add the admin role to the glance user and service project

```
    openstack role add --project service --user glance admin
```

- Create the glance service entity: 

```
    openstack service create --name glance --description "OpenStack Image" image
```

- Create the Image service API endpoints: 

```
    openstack endpoint create --region RegionOne image public http://ngoc-ctl:9292

    openstack endpoint create --region RegionOne image internal http://ngoc-ctl:9292

    openstack endpoint create --region RegionOne image admin http://ngoc-ctl:9292
```

## Install and configure components 

- Install the packages:

```
apt install glance
```

- Edit the /etc/glance/glance-api.conf file and complete the following actions:

```
[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@ngoc-ctl/glance
Replace GLANCE_DBPASS with the password you chose for the Image service database.

[keystone_authtoken]
www_authenticate_uri = http://ngoc-ctl:5000
auth_url = http://ngoc-ctl:5000
memcached_servers = ngoc-ctl:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

- Populate the Image service database:

```
    su -s /bin/sh -c "glance-manage db_sync" glance
```

## Finalize installation 

- Restart the Image services: 

```
    service glance-api restart
```

## Verify operation 

```
    . admin-openrc 

    wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img

    glance image-create --name "cirros" \
        --file cirros-0.4.0-x86_64-disk.img \
        --disk-format qcow2 --container-format bare \
        --visibility=public

    glance image-list
```