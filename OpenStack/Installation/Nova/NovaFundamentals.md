## Nova 
OpenStack Compute is a major part of an Infrastructure-as-a-Service (IaaS) system. It gives you control over instances and networks, and allows you to manage access to the cloud through users and projects. 

OpenStack Compute interacts with OpenStack Identity for authentication, OpenStack Placement for resource inventory tracking and selection, OpenStack Image service for disk and server images, and OpenStack Dashboard for the user and administrative interface. Image access is limited by projects, and by users; quotas are limited per project (the number of instances, for example). OpenStack Compute can scale horizontally on standard hardware, and download images to launch instances.

OpenStack Compute does not have virtualization software. Instead, it defines drivers that interact with underlying virtualization mechanisms that run on your host operating system, and exposes functionality over a web-based API. 

OpenStack Compute consists of the following areas and their components:
- **nova-api (service)**: Accepts and responds to end user compute API calls. The service supports the OpenStack Compute API. It enforces some policies and initiates most orchestration activities, such as running an instance.

- **nova-api-metadata (service)**: Accepts metadata requests from instances e.g. ip, hostname, ssh keys, cloud-init script. (Refer to [this link](https://docs.openstack.org/nova/victoria/admin/metadata-service.html)).

- **nova-compute (service)**: A worker daemon that creates and terminates virtual machine instances through hypervisor APIs. For example:
+ libvirt for KVM or QEMU
+ VMwareAPI for VMware
Processing is fairly complex. Basically, the daemon accepts actions from the queue and performs a series of system commands such as launching a KVM instance and updating its state in the database. 
Each compute node needs to have a separate nova-compute installation. 

- **nova-scheduler (service)**: Takes a virtual machine instance request from the queue and determines on which compute server host it runs.

- **nova-conductor (module)**: Mediates interactions between the nova-compute service and the database. It eliminates direct accesses to the cloud database made by the nova-compute service. The nova-conductor module scales horizontally. However, do not deploy it on nodes where the nova-compute service runs. 

- **nova-novncproxy (daemon)**: Provides a proxy for accessing running instances through a VNC connection. Supports browser-based novnc clients.

- **nova-spicehtml5proxy (daemon)**: Provides a proxy for accessing running instances through a SPICE connection. Supports browser-based HTML5 client.

- **The queue**: A central hub for passing messages between daemons. Usually implemented with RabbitMQ. 

- **SQL database**: Stores most build-time and run-time states for a cloud infrastructure, including:
+ Available instance types
+ Instances in use
+ Available networks
+ Projects
Theoretically, OpenStack Compute can support any database that SQLAlchemy supports. Common databases are SQLite3 for test and development work, MySQL, MariaDB, and PostgreSQL.

[Component diagram](https://docs.openstack.org/nova/pike/_images/architecture.svg)

### Nova API 

First, there is the Nova API server which runs on the controller node. The Nova service will register itself as a systemd service with entry point /usr/bin/nova-api. Similar to Glance, invoking this script will bring up an WSGI server which uses PasteDeploy to build a pipeline with the actual Nova API endpoint (an instance of nova.api.openstack.compute.APIRouterV21) being the last element of the pipeline. This component will then distribute incoming API requests to various controllers which are part of the nova.api.openstack.compute module. The routing rules themselves are actually hardcoded in the ROUTE_LIST which is part of the Router class and maps request paths to controller objects and their methods.

When you browse the source code, you will find that Nova offers some APIs like the image API or the bare metal API which are simply proxies to other OpenStack services like Glance or Ironic.

### Nova conductor 
The Nova conductor does not expose a REST API, but communicates with the other Nova components via RPC calls (based on RabbitMQ). The conductor is used to handle long-running tasks like building an instance or performing a live migration.

Similar to the Nova API server, the conductor has a tiered architecture. The actual binary which is started by the systemd mechanism creates a so called service object. In Nova, a service objects represents an RPC API endpoint. When a service object is created, it starts up an RPC service that handles the actual communication via RabbitMQ and forwards incoming requests to an associated service manager object.

[https://leftasexercise.files.wordpress.com/2019/11/novarpcapi-1.png](https://leftasexercise.files.wordpress.com/2019/11/novarpcapi-1.png)

SERVICE_MANAGERS = {
  'nova-compute': 'nova.compute.manager.ComputeManager',
  'nova-console': 'nova.console.manager.ConsoleProxyManager',
  'nova-conductor': 'nova.conductor.manager.ConductorManager',
  'nova-metadata': 'nova.api.manager.MetadataManager',
  'nova-scheduler': 'nova.scheduler.manager.SchedulerManager',
}

### Nova scheduler 

The scheduler receives and maintains information on the instances running on the individual hosts and, upon request, uses the Placement API that we have looked at in the previous post to take a decision where a new instance should be placed. The actual scheduling is carried out by a pluggable instance of the nova.scheduler.Scheduler base class. The default scheduler is the filter scheduler which first applies a set of filters to filter out individual hosts which are candidates for hosting the instance, and then computes a score using a set of weights to take a final decision. This is also used to choose a new host when migrating. 

Compute is configured with the following default scheduler options in the /etc/nova/nova.conf file:

```
[scheduler]
driver = filter_scheduler

[filter_scheduler]
available_filters = nova.scheduler.filters.all_filters
enabled_filters = RetryFilter, AvailabilityZoneFilter, ComputeFilter, ComputeCapabilitiesFilter, ImagePropertiesFilter, ServerGroupAntiAffinityFilter, ServerGroupAffinityFilter
```

By default, the scheduler driver is configured as a filter scheduler, as described in the next section. In the default configuration, this scheduler considers hosts that meet all the following criteria:

- Have not been attempted for scheduling purposes (RetryFilter).
- Are in the requested availability zone (AvailabilityZoneFilter).
- Can service the request (ComputeFilter).
- Satisfy the extra specs associated with the instance type (ComputeCapabilitiesFilter).
- Satisfy any architecture, hypervisor type, or virtual machine mode properties specified on the instance’s image properties (ImagePropertiesFilter).
- Are on a different host than other instances of a group (if requested) (ServerGroupAntiAffinityFilter).
- Are in a set of group hosts (if requested) (ServerGroupAffinityFilter).


#### Filtering 

The filter scheduler (nova.scheduler.filter_scheduler.FilterScheduler) is the default scheduler for scheduling virtual machine instances. It supports filtering and weighting to make informed decisions on where a new instance should be created.

When the filter scheduler receives a request for a resource, it first applies filters to determine which hosts are eligible for consideration when dispatching a resource. Filters are binary: either a host is accepted by the filter, or it is rejected. Hosts that are accepted by the filter are then processed by a different algorithm to decide which hosts to use for that request, described in the Weights section.

[https://docs.openstack.org/nova/rocky/_images/filteringWorkflow1.png](https://docs.openstack.org/nova/rocky/_images/filteringWorkflow1.png)

There are many options to filtering: based in CPU core numbers, disk allocation, tentant creating all instances only on specific Host aggregates and availability zones. 

If scheduler cannot choose any hosts then that instance could not be scheduled. 

#### Weighting 

When resourcing instances, the filter scheduler filters and weights each host in the list of acceptable hosts. Each time the scheduler selects a host, it virtually consumes resources on it, and subsequent selections are adjusted accordingly. This process is useful when the customer asks for the same large amount of instances, because weight is computed for each requested instance.

All weights are normalized (with a configurable multiplier) before being summed up; the host with the largest weight is given the highest priority.

There are many options to host weighting. 

[https://docs.openstack.org/nova/rocky/_images/nova-weighting-hosts.png](https://docs.openstack.org/nova/rocky/_images/nova-weighting-hosts.png)

In general, scheduling in nova depends on the nova.conf file, service capability of each host and request specifications. 

### Nova compute 

One instance of the Nova compute service runs on each compute node. The manager class behind this service is the ComputeManager which itself invokes various APIs like the networking API or the Cinder API to manage the instances on this node. The compute service interacts with the underlying hypervisor via a compute driver. Nova comes with compute driver for the most commonly used hypervisors, including KVM (via libvirt), VMWare, HyperV or the Xen hypervisor. 

The Nova compute service itself does not have a connection to the database. However, in some cases, the compute service needs to access information stored in the database, for instance when the Nova compute service initializes on a specific host and needs to retrieve a list of instances running on this host from the database. To make this possible, Nova uses remotable objects provided by the Oslo versioned objects library. This library provides decorators like remotable_classmethod to mark methods of a class or an object as remotable. These decorators point to the conductor API (indirection_api within Oslo) and delegate the actual method invocation to a remote copy via an RPC call to the conductor API. In this way, only the conductor needs access to the database and Nova compute offloads all database access to the conductor.

### Nova metadata 

[Diagram](https://tuantuluong.files.wordpress.com/2014/09/neutron-metadata-dhcp-agent.png)
[Diagram 2](https://ibm-blue-box-help.github.io/help-documentation/img/en/Openstack_metadata.png)

### Nova cells 

In a large OpenStack installation, access to the instance data stored in the MariaDB database can easily become a bottleneck. To avoid this, OpenStack provides a sharding mechanism for the database known as cells.

The idea behind this is that the set of your compute nodes are partitioned into cells. Every compute node is part of a cell, and in addition to these regular cells, there is a cell called cell0 (which is usually not used and only holds instances which could not be scheduled to a node). The Nova database schema is split into a global part which is stored in a database called the API database and a cell-local part. This cell-local database is different for each cell, so each cell can use a different database running (potentially) on a different host. A similar sharding applies to message queues. When you set up a compute node, the configuration of the database connection and the connection to the RabbitMQ service determine to which cell the node belongs. The compute node will then use this database connection to register itself with the corresponding cell database, and a special script (nova-manage) needs to be run to make these hosts visible in the API database as well so that they can be used by the scheduler.

Cells themselves are stored in a database table cell_mappings in the API database. Here each cell is set up with a dedicated RabbitMQ connection string (called the transport URL) and a DB connection string. Our setup will have two cells – the special cell0 which is always present and a cell1. Therefore, our installation will required three databases.

data base      description
nova_api       nova API database
nova_cell0     database for cell0
nova           database for cell1

[https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenStack_Platform-16.0-Instances_and_Images_Guide-en-US/images/114e111c687c366efe2f8b083adcf837/multi-cell-deployment.png](https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenStack_Platform-16.0-Instances_and_Images_Guide-en-US/images/114e111c687c366efe2f8b083adcf837/multi-cell-deployment.png)

### Nova VNC proxy 
Usually, you will use SSH to access your instances. However, sometimes, for instance if the SSHD is not coming up properly or the network configuration is broken, it would be very helpful to have a way to connect to the instance directly. For that purpose, OpenStack offers a VNC console access to running instances. Several VNC clients can be used, but the default is to use the noVNC browser based client embedded directly into the Horizon dashboard.

How exactly does this work? First, there is KVM. The KVM hypervisor has the option to export the content of the emulated graphics card of the instance as a VNC server. Obviously, this VNC server is running on the compute node on which the instance is located. The server for the first instance will listen on port 5900, the server for the second instance will listen on port 5901 and so forth. The server_listen configuration option determines the IP address to which the server will bind.

Now theoretically a VNC client like noVNC could connect directly to the VNC server. However, in most setups, the network interfaces of the compute node are not directly reachable from a browser in which the Horizon GUI is running. To solve this, Nova comes with a dedicated proxy for noVNC. This proxy is typically running on the controller node. The IP address and port number on which this proxy is listening can again be configured using the novncproxy_host and novncproxy_port configuration items. The default port is 6080.

When a client like the Horizon dashboard wants to get access to the proxy, it can use the Nova API path /servers/{server_id}/remote-consoles. This call will be forwarded to the Nova compute method get_vnc_console on the compute node. This method will return an URL, consisting of the base URL (which can again be configured using the novncproxy_base_url configuration item), and a token which is stored in the database as well. When the client uses this URL to connect to the proxy, the token is used to verify that the call is authorized.

[https://leftasexercise.files.wordpress.com/2019/11/novncproxy.png](https://leftasexercise.files.wordpress.com/2019/11/novncproxy.png)

### Resizing in OpenStack 
Resizing is the technique to resize the instance by changing its flavour. 

Command for resizing: 

```
openstack server resize --flavor <flavor_name> <instance_name>
```

Comamnd for confirming it is working: 

```
openstack server resize confirm <instance_name>
```

If resizing does not work , we can revert back to its old flavour: 

```
openstack server resize revert <instance_name>
```

### Rescuing in OpenStack 

Instance rescue provides a mechanism for access, even if an image renders the instance inaccessible. Two rescue modes are currently provided.

Should a user lose an SSH key, or be otherwise not be able to boot and access an instance, say, bad IPTABLES settings or failed network configuration, rescue mode will start a minimal instance and attach the disk from the failed instance to aid in recovery.

When put into a rescue state, your image will be rebooted using a rescue image (i.e. the rescue image is mounted as the first boot source). This rescue image should not contain the problem configurations that are causing issues for this instance when booted normally. This means that you can boot and connect to your rescued image, allowing you to correct the problems that are preventing you operating it normally. After correcting these problems, you can revert your image back to its normal active state.

#### Instance rescue 

By default the instance is booted from the provided rescue image or a fresh copy of the original instance image if a rescue image is not provided. The root disk and optional regenerated config drive are also attached to the instance for data recovery.
Rescuing a volume-backed instance is not supported with this mode.

#### Stable device instance rescue 
This mode now supports the rescue of volume-backed instances.

This mode keeps all devices both local and remote attached in their original order to the instance during the rescue while booting from the provided rescue image. This mode is enabled and controlled by the presence of hw_rescue_device or hw_rescue_bus image properties on the provided rescue image.

To perform an instance rescue, use the openstack server rescue command:

```
openstack server rescue SERVER
```

Pause, suspend, and stop operations are not allowed when an instance is running in rescue mode, as triggering these actions causes the loss of the original instance state and makes it impossible to unrescue the instance.

On running the openstack server rescue command, an instance performs a soft shutdown first. This means that the guest operating system has a chance to perform a controlled shutdown before the instance is powered off. The shutdown behavior is configured by the shutdown_timeout parameter that can be set in the nova.conf file. Its value stands for the overall period (in seconds) a guest operating system is allowed to complete the shutdown.

To restart the instance from the normal boot disk, run the following command:

```
openstack server unrescue SERVER
```

### Evacuation 

If a hardware malfunction or other error causes a cloud compute node to fail, you can evacuate instances to make them available again.

To preserve user data on the server disk, configure shared storage on the target host. When you evacuate the instance, Compute detects whether shared storage is available on the target host. Also, you must validate that the current VM host is not operational. Otherwise, the evacuation fails.

Only works if the instance is using shared storage or block storage, otherwise the hard disk cannot be accessed from external sources in case the host failed. 

There are two different ways to evacuate instances from a failed compute node. The first one using the nova evacuate command can be used to evacuate a single instance from a failed node. In some cases where the node in question hosted many instances it might be easier to use nova host-evacuate to evacuate them all in one shot.

https://wiki.openstack.org/w/images/6/6b/Evacuate_instance.png


### How Nova uses Placement 
Two processes, nova-compute and nova-scheduler, host most of nova’s interaction with placement.

The nova resource tracker in nova-compute is responsible for creating the resource provider record corresponding to the compute host on which the resource tracker runs, setting the inventory that describes the quantitative resources that are available for workloads to consume (e.g., VCPU), and setting the traits that describe qualitative aspects of the resources (e.g., STORAGE_DISK_SSD).

The nova-scheduler is responsible for selecting a set of suitable destination hosts for a workload. It begins by formulating a request to placement for a list of allocation candidates. That request expresses quantitative and qualitative requirements, membership in aggregates, and in more complex cases, the topology of related resources. That list is reduced and ordered by filters and weighers within the scheduler process. An allocation is made against a resource provider representing a destination, consuming a portion of the inventory set by the resource tracker.

For Placement API, in each request we need to include two header parameters: 
- X-Auth-Token needs to contain a valid token that we need to retrieve from Keystone first
- OpenStack-API-Version needs to be included to define the version of the API (this is the so-called microversion).

### Launch an instance 

From nova-scheduler -> nova-conductor -> write to database updating states. 

[Diagram 1](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Nova/novaimg/nova-cre-ins1.png)
[Diagram 2](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Nova/novaimg/nova-cre-ins2.png)

! Note: 
- When launching an instance not as an admin, you can not force or choose host to launch instances on. 
- Specify Host as Scheduler Hint: Add a scheduler hint with the key force_host and set its value to the name of the specific host where you want to launch the instance.
Example: Key: force_host, Value: my-target-host

#### How this instance communicates with an external host 

- The instance generates a packet and places it on the virtual Network Interface Card (NIC) inside the instance, such as eth0.

- The packet transfers to the virtual NIC of the compute host, such as, vnet1. You can find out what vnet NIC is being used by looking at the /etc/libvirt/qemu/instance-xxxxxxxx.xml file.

- From the vnet NIC, the packet transfers to a bridge on the compute node, such as br100.

- If you run FlatDHCPManager, one bridge is on the compute node. If you run VlanManager, one bridge exists for each VLAN.

- The packet transfers to the main NIC of the compute node. You can also see this NIC in the brctl output, or you can find it by referencing the flat_interface option in nova.conf.

- After the packet is on this NIC, it transfers to the compute node’s default gateway. The packet is now most likely out of your control at this point. The diagram depicts an external gateway. However, in the default configuration with multi-host, the compute host is the gateway.

- Reverse the direction to see the path of a ping reply. From this path, you can see that a single packet travels across four different NICs. If a problem occurs with any of these NICs, a network issue occurs.


### Useful links
Command-line utilities
https://docs.openstack.org/api-ref/placement/ 
Configuration guide
https://docs.openstack.org/placement/victoria/configuration/index.html 

### Key terms
Microversioning: https://specs.openstack.org/openstack/api-sig/guidelines/microversion_specification.html 

Availibility zone: are an end-user visible logical abstraction for partitioning a cloud without knowing the physical infrastructure, often used to partition a cloud based on location. 
Host aggregate: Host aggregates are a mechanism for partitioning hosts in an OpenStack cloud, or a region of an OpenStack cloud, based on arbitrary characteristics. Examples where an administrator may want to do this include where a group of hosts have additional hardware or performance characteristics. One host can be in multiple aggregates (only visible to administrators)

Oslo concurrency: library has utilities for safely running multi-thread, multi-process applications using locking mechanisms and for running external processes.

libvirt: an open-source API, daemon and management tool for managing platform virtualization. It can be used to manage KVM, Xen, VMware ESXi, QEMU and other virtualization technologies. These APIs are widely used in the orchestration layer of hypervisors in the development of a cloud-based solution.
 
Flavor: specifies a virtual resource allocation profile which includes processor, memory, and storage.

KVM hypervisor: is an open-source virtualization technology that allows running multiple virtual machines (VMs) on a host machine. It is a hypervisor, which is a software layer that enables the creation and management of virtualized environments (type 1).

QEMU vs KVM hypervisor: KVM Kernel-based Virtual Machine) is the type 1 hypervisor built into every Linux kernel since version 2.6.20. QEMU and is one of example of a type 2 hypervisor that can utilize KVM to allow virtual machines direct access to your system’s hardware. It is possible to use KVM directly, without the need for a type 2 hypervisor, but doing so is much more complex and not very user friendly. In scenarios of critical production servers that are overseen by seasoned administrators, they will often utilize KVM without the type 2 hypervisor interface at all.

noVNC-based VNC console: VNC is a graphical console with wide support among many hypervisors and clients. noVNC provides VNC support through a web browser.

cloud-init script: cloud-init is a software package that automates the initialization of cloud instances during system boot. You can configure cloud-init to perform a variety of tasks. 

virsh: The virsh program is the main interface for managing virsh guest domains. The program can be used to create, pause, and shutdown domains. It can also be used to list current domains. Libvirt is a C toolkit to interact with the virtualization capabilities of recent versions of Linux (and other OSes). It is free software available under the GNU Lesser General Public License. Virtualization of the Linux Operating System means the ability to run multiple instances of Operating Systems concurrently on a single hardware system where the basic resources are driven by a Linux instance. The library aims at providing a long term stable C API . It currently supports Xen, QEmu, KVM , LXC , OpenVZ, VirtualBox and VMware ESX .


### References
(https://docs.openstack.org/nova/victoria/install/overview.html)[https://docs.openstack.org/nova/victoria/install/overview.html] 
https://leftasexercise.com/2020/02/14/openstack-nova-installation-and-overview/
https://docs.openstack.org/nova/latest/admin/cells.html
https://www.civo.com/learn/putting-your-instance-into-rescue-mode
https://blog.codybunch.com/2017/09/08/Rescuing-an-OpenStack-instance/
https://docs.openstack.org/nova/rocky/admin/evacuate.html
https://docs.openstack.org/nova/rocky/admin/configuration/schedulers.html