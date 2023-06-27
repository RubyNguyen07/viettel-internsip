This section describes how to install and configure the OpenStack Identity service, code-named keystone, on the controller node. For scalability purposes, this configuration deploys Fernet tokens and the Apache HTTP server to handle requests.


## Prerequisites
Before you install and configure the Identity service, you must create a database.

- Use the database access client to connect to the database server as the root user:

```
    mysql
```

- Create the keystone database and grant proper access to the keystone database:

```
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
```

- Exit the database access client.

## Install and configure components 

- Run the following command to install the packages:

```
    apt install keystone apache2 libapache2-mod-wsgi-py3 python3-oauth2client
```

- Edit the /etc/keystone/keystone.conf file and complete the following actions:

    + In the [database] section, configure database access:

```
    [database]
    connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
```

    + In the [token] section, configure the Fernet token provider:

```
    [token]
    provider = fernet
```

- Populate the Identity service database:

```
    su -s /bin/sh -c "keystone-manage db_sync" keystone
```

- Initialize Fernet key repositories:

```
    keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
    keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

- Bootstrap the Identity service:

```
 keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

## Configure the Apache HTTP server
- Edit the /etc/apache2/apache2.conf file and configure the ServerName option to reference the controller node:

```
    ServerName ngoc-ctl
```

## Finalize the installation

- Restart the Apache service:

```
    service apache2 restart
```

- Configure the administrative account by setting the proper environmental variables:

```
$ export OS_USERNAME=admin
$ export OS_PASSWORD=ADMIN_PASS
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=Default
$ export OS_PROJECT_DOMAIN_NAME=Default
$ export OS_AUTH_URL=http://ngoc-ctl:5000/v3
$ export OS_IDENTITY_API_VERSION=3
```
These values shown here are the default ones created from keystone-manage bootstrap.

Replace ADMIN_PASS with the password used in the keystone-manage bootstrap command. 


Keywords: 
- WSGI: WSGI is a specification that describes the communication between web servers and Python web applications or frameworks. It explains how a web server communicates with python web applications/frameworks and how web applications/frameworks can be chained for processing a request.
https://miro.medium.com/v2/resize:fit:1400/format:webp/1*BN2BMqbzlignUJZiX7Xxwg.png 


## Verify operation

```
    unset OS_AUTH_URL OS_PASSWORD
    openstack --os-auth-url http://ngoc-ctl:5000/v3 \
    --os-project-domain-name Default --os-user-domain-name Default \
    --os-project-name admin --os-username admin token issue
```



References: 
- https://www.server-world.info/en/note?os=Ubuntu_20.04&p=openstack_yoga&f=3
