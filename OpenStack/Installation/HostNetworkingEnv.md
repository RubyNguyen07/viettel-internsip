> This is an installation OpenStack:Victoria guide on Ubuntu. 
  Here is the configurations on my controller node: 
    Hostname = ngoc-ctl 
    First interface (eth0) = 192.168.30.39 (this IP has been configured to access the Internet)
    Second interface (eth1) = 192.168.16.161
    Netmask = 255.255.255.0

   Here is the configurations on my compute node: 
    Hostname = ngoc-com 
    First interface (eth0) = 192.168.30.143 (this IP has been configured to access the Internet)
    Second interface (eth1) = 192.168.16.129
    Netmask = 255.255.255.0

    Here is the configurations on my storage node: 
    Hostname = ngoc-sto 
    First interface (eth0) = 192.168.30.177 (this IP has been configured to access the Internet)
    Second interface (eth1) = 192.168.16.86
    Netmask = 255.255.255.0

    Python version = 3.8.10


## Host Networking 
This step happens after installing the operating system on each node for the architecture that you choose to deploy. 
For the two nodes, I will configure two network interfaces for each, one of them is to access the Internet. Note that 
for all nodes, there needs to be at least 1 network interface to connect to the Internet (for package installation, 
security update, ...)

### Controller Node 
#### Configure network interfaces 
- Configure the first interface as the provider interface (in /etc/network/interfaces file)
    auto eth0 -> this means interface eth0 should be configured during boot time, and the name is eth0
    iface eth0 inet static -> this defines a static IP address for eth0 (as we have already known this IP, otherwise can set static to dhcp to receive an IP address)
    address 192.168.30.39 -> this is the IP address we have known 
    netmask 255.255.255.0 
    gateway 192.168.30.1

- Configure the second interface as the management interface (in /etc/network/interfaces file)
    auto eth1
    iface eth1 inet static
    address 192.168.16.161
    netmask 255.255.255.0
    gateway 192.168.16.1

- Reboot the system to activate the changes: sudo /etc/init.d/networking restart </li>

``` 
    sudo /etc/init.d/networking restart
```



#### Configure name resolution 

- Set the hostname of the node to whatever you like (mine was set to ngoc-ctl) 
- Edit the /etc/hosts file to contain the following:

```
    # controller
    192.168.30.39      ngoc-ctl
    
    # compute
    192.168.30.143      ngoc-com

    # block storage 
    192.168.30.177      ngoc-sto
```



### Compute Node 
#### Configure network interfaces 

- Configure the first interface as the provider interface (in /etc/network/interfaces file)
    auto eth0 -> this means interface eth0 should be configured during boot time, and the name is eth0
    iface eth0 inet static -> this defines a static IP address for eth0 (as we have already known this IP, otherwise can set static to dhcp to receive an IP address)
    address 192.168.30.143 -> this is the IP address we have known 
    netmask 255.255.255.0 
    gateway 192.168.30.1

- Configure the second interface as the management interface (in /etc/network/interfaces file)
    auto eth1
    iface eth1 inet static
    address 192.168.16.129
    netmask 255.255.255.0
    gateway 192.168.16.1

- Reboot the system to activate the changes: sudo /etc/init.d/networking restart 

```console 
    sudo /etc/init.d/networking restart
```


#### Configure name resolution 
<ol>
    <li> Set the hostname of the node to whatever you like (mine was set to ngoc-com) </li>
    <li> Edit the /etc/hosts file to contain the following: </li>

    ```
       # controller
       192.168.30.39      ngoc-ctl
       
       # compute
       192.168.30.143      ngoc-com

       # block storage 
       192.168.30.177      ngoc-sto
    ```
</ol> 

### Block Storage Node 

#### Configure name resolution 

- Set the hostname of the node to whatever you like (mine was set to ngoc-sto) 
- Edit the /etc/hosts file to contain the following:

```
    # controller
    192.168.30.39      ngoc-ctl
    
    # compute
    192.168.30.143      ngoc-com

    # block storage 
    192.168.30.177      ngoc-sto
```

- Reboot the system to activate the changes. 

### Verify connectivity 
After the above steps, you need to verify network connectivity to the Internet and among the nodes: 

- From the controller node, test access to Internet.

``` 
    ping docs.openstack.org 
```

- From the controller node, test access to the management interface on the compute node.

``` 
    ping -c 4 ngoc-com 
```

- From the compute node, test access to Internet.

``` 
    ping docs.openstack.org 
```

- From the compute node, test access to the management interface on the controller node.

``` 
    ping -c 4 ngoc-ctl 
```



## Network Time Protocol (NTP)
**Usage**: This setup is for synchronizing services among nodes. You should configure the controller node to reference more 
accurate servers and other nodes to reference the controller node. In this guide, I will use Chrony. 

### Controller Node 
#### Install and configure components 
- Install the packages 

``` 
    apt install chrony 
```

- Add the following key in the /etc/chrony/chrony.conf file after the lines starting with pool (server from the line with highest line number will be chosen to be the NTP server)

``` 
    server 0.vn.pool.ntp.org
    server 0.asia.pool.ntp.org
    server 2.asia.pool.ntp.org
    
```

- Replace NTP_SERVER with the hostname or IP address of a NTP server (choose one with your location's time for easier debugging later on)

- To enable other nodes to connect to the chrony daemon on the controller node, add this key to the same chrony.conf file mentioned above: 

```
    allow 192.168.30.0/24
```

- Restart the NTP service: 

```
    service chrony restart  
```


### Compute Node 
#### Install and configure components 
- Install the packages 

``` 
    apt install chrony 
```

- Configure the /etc/chrony/chrony.conf file and comment out or remove all server keys. Change it to reference the controller node by adding the following line.

```
    server ngoc-ctl iburst
```

- Restart the NTP service: 

```console
    service chrony restart  
```


### Storage Node 
#### Install and configure components 
- Install the packages 

``` 
    apt install chrony 
```

- Configure the /etc/chrony/chrony.conf file and comment out or remove all server keys. Change it to reference the controller node by adding the following line.

```
    server ngoc-ctl iburst
```

- Restart the NTP service: 

```console
    service chrony restart  
```


### Verify operation 
- Run the below command on the controller node. Contents in the Name/IP address column should indicate the hostname or IP address of one or more NTP servers. Contents in the MS column should indicate * for the server to which the NTP service is currently synchronized.

``` 
    chronyc sources
```

- Run the same command on the compute and storage node. Contents in the Name/IP address column should indicate the hostname of the controller node.



## OpenStack packages for Ubuntu 

- Configure OpenStack Victoria repository on all nodes that run OpenStack services (here I will configure on controller and compute nodes).

```
add-apt-repository cloud-archive:victoria
apt update 
apt -y upgrade 
apt install python3-openstackclient
```

## SQL database 

**Usage**: most OpenStack services use an SQL database to store information. 

Complete the following actions on the controller node. 

- Install the packages: 

```
apt install mariadb-server python3-pymysql
```

- Create and edit the /etc/mysql/mariadb.conf.d/99-openstack.cnf file. Create a [mysqld] section, and set the bind-address key to the management IP address of the controller node to enable access by other nodes via the first interface. Set additional keys to enable useful options and the UTF-8 character set:

```
[mysqld]
bind-address = 192.168.30.39

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

- Restart the database service: 

```
    service mysql restart
```

- Secure the database service by running the mysql_secure_installation script. In particular, choose a suitable password for the database root account:

```
    mysql_secure_installation
```

## Message queue 

**Usage**: OpenStack uses a message queue to coordinate operations and status information among services. 

Complete the following actions on the controller node: 

- Install the package: 

```
apt install rabbitmq-server
```

- Add the OpenStack user: 

```
rabbitmqctl add_user openstack RABBIT_PASS
```

- Permit configuration, write, and read access for the openstack user:

```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## Memcached 

**Usage**: The Identity service authentication mechanism for services uses Memcached to cache tokens. 

Complete the following actions on the controller node: 

- Install the packages: 

```
apt install memcached python3-memcache
```

- Edit the /etc/memcached.conf file and configure the service to use the first interface of the controller node. This is to enable access by other nodes via the first interface:

```
-l 192.168.30.39 
```

- Restart the Memcached service:

```
    service memcached restart 
```

## Etcd 

**Usage**: OpenStack services may use Etcd, a distributed reliable key-value store for distributed key locking, storing configuration, keeping track of service live-ness and other scenarios.

Complete the following actions on the controller node: 

- Install the etcd package:

```
apt install etcd
```

- Edit the /etc/default/etcd file and set the ETCD_INITIAL_CLUSTER, ETCD_INITIAL_ADVERTISE_PEER_URLS, ETCD_ADVERTISE_CLIENT_URLS, ETCD_LISTEN_CLIENT_URLS to the first interface the controller node: 

```
ETCD_NAME="ngoc-ctl"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="ngoc-ctl=http://192.168.30.39:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.30.39:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.30.39:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.30.39:2379"
```

- Enable and restart the etcd service:

```
    systemctl enable etcd
    systemctl restart etcd
```


## References 
[https://docs.openstack.org/install-guide/environment-networking.html](https://docs.openstack.org/install-guide/environment-networking.html)
[https://www.tecmint.com/ip-command-examples/](https://www.tecmint.com/ip-command-examples/)
[https://www.cyberciti.biz/faq/linux-restart-network-interface/](https://www.cyberciti.biz/faq/linux-restart-network-interface/)