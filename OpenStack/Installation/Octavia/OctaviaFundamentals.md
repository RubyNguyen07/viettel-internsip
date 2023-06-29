## Octavia 

Octavia is an open source, operator-scale load balancing solution designed to work with OpenStack.

Octavia was born out of the Neutron LBaaS project. Its conception influenced the transformation of the Neutron LBaaS project, as Neutron LBaaS moved from version 1 to version 2. Starting with the Liberty release of OpenStack, Octavia has become the reference implementation for Neutron LBaaS version 2.

Octavia accomplishes its delivery of load balancing services by managing a fleet of virtual machines, containers, or bare metal servers—collectively known as amphorae— which it spins up on demand. This on-demand, horizontal scaling feature differentiates Octavia from other load balancing solutions, thereby making Octavia truly suited “for the cloud”.

### Where Octavia fits into the OpenStack ecosystem 

Load balancing is essential for enabling simple or automatic delivery scaling and availability. In turn, application delivery scaling and availability must be considered vital features of any cloud. Together, these facts imply that load balancing is a vital feature of any cloud.

Therefore, we consider Octavia to be as essential as Nova, Neutron, Glance or any other “core” project that enables the essential features of a modern OpenStack cloud.

In accomplishing its role, Octavia makes use of other OpenStack projects:

- Nova: For managing amphora lifecycle and spinning up compute resources on demand.

- Neutron: For network connectivity between amphorae, tenant environments, and external networks.

- Barbican: For managing TLS certificates and credentials, when TLS session termination is configured on the amphorae.

- Keystone: For authentication against the Octavia API, and for Octavia to authenticate with other OpenStack projects.

- Glance: For storing the amphora virtual machine image.

- Oslo: For communication between Octavia controller components, making Octavia work within the standard OpenStack framework and review system, and project code structure.

- Taskflow: Is technically part of Oslo; however, Octavia makes extensive use of this job flow system when orchestrating back-end service configuration and management.

Octavia is designed to interact with the components listed previously. In each case, we’ve taken care to define this interaction through a driver interface. That way, external components can be swapped out with functionally-equivalent replacements — without having to restructure major components of Octavia. For example, if you use an SDN solution other than Neutron in your environment, it should be possible for you to write an Octavia networking driver for your SDN environment, which can be a drop-in replacement for the standard Neutron networking driver in Octavia.


### A 10,000-foot overview of Octavia components

- Amphorae - Amphorae are the individual virtual machines, containers, or bare metal servers that accomplish the delivery of load balancing services to tenant application environments. In Octavia version 0.8, the reference implementation of the amphorae image is an Ubuntu virtual machine running HAProxy.

- Controller - The Controller is the “brains” of Octavia. It consists of five sub-components, which are individual daemons. They can be run on separate back-end infrastructure if desired:

+ API Controller - As the name implies, this subcomponent runs Octavia’s API. It takes API requests, performs simple sanitizing on them, and ships them off to the controller worker over the Oslo messaging bus.

+ Controller Worker - This subcomponent takes sanitized API commands from the API controller and performs the actions necessary to fulfill the API request.

+ Health Manager - This subcomponent monitors individual amphorae to ensure they are up and running, and otherwise healthy. It also handles failover events if amphorae fail unexpectedly.

+ Housekeeping Manager - This subcomponent cleans up stale (deleted) database records and manages amphora certificate rotation.

+ Driver Agent - The driver agent receives status and statistics updates from provider drivers.

- network - Octavia cannot accomplish what it does without manipulating the network environment. Amphorae are spun up with a network interface on the “load balancer network,” and they may also plug directly into tenant networks to reach back-end pool members, depending on how any given load balancing service is deployed by the tenant.



## Keywords 

- Amphora: Virtual machine, container, dedicated hardware, appliance or device that actually performs the task of load balancing in the Octavia system. More specifically, an amphora takes requests from clients on the front-end and distributes these to back-end systems. Amphorae communicate with their controllers over the LB Network through a driver interface on the controller.

- VIP (Virtual IP address): Single service IP address which is associated with a load balancer. This is similar to what is described here: http://en.wikipedia.org/wiki/Virtual_IP_address In a highly available load balancing topology in Octavia, the VIP might be assigned to several amphorae, and a layer-2 protocol like CARP, VRRP, or HSRP (or something unique to the networking infrastructure) might be used to maintain its availability. In layer-3 (routed) topologies, the VIP address might be assigned to an upstream networking device which routes packets to amphorae, which then load balance requests to back-end members.

- TLS Termination (Transport Layer Security Termination): Type of load balancing protocol where HTTPS sessions are terminated (decrypted) on the amphora as opposed to encrypted packets being forwarded on to back-end servers without being decrypted on the amphora. Also known as SSL termination. The main advantages to this type of load balancing are that the payload can be read and / or manipulated by the amphora, and that the expensive tasks of handling the encryption are off-loaded from the back-end servers. This is particularly useful if layer 7 switching is employed in the same listener configuration. 

- LB Network (Load Balancer Network): The network over which the controller(s) and amphorae communicate. The LB network itself will usually be a nova or neutron network to which both the controller and amphorae have access, but is not associated with any one tenant. The LB Network is generally also not part of the undercloud and should not be directly exposed to any OpenStack core components other than the Octavia Controller.

- HAProxy: Load balancing software used in the reference implementation for Octavia. (See http://www.haproxy.org/ ). HAProxy processes run on amphorae and actually accomplish the task of delivering the load balancing service.





## Resources 
https://docs.openstack.org/octavia/latest/reference/introduction.html
https://docs.openstack.org/octavia/latest/reference/glossary.html