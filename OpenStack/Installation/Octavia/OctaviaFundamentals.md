## Octavia 

Octavia is an open source, operator-scale load balancing solution designed to work with OpenStack.

Octavia was born out of the Neutron LBaaS project. Its conception influenced the transformation of the Neutron LBaaS project, as Neutron LBaaS moved from version 1 to version 2. Starting with the Liberty release of OpenStack, Octavia has become the reference implementation for Neutron LBaaS version 2.

Octavia accomplishes its delivery of load balancing services by managing a fleet of virtual machines, containers, or bare metal servers—collectively known as amphorae — which it spins up on demand. This on-demand, horizontal scaling feature differentiates Octavia from other load balancing solutions, thereby making Octavia truly suited “for the cloud”.

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

### Examples of using Octavia 

![Diagram](https://access.redhat.com/webassets/avalon/d/Red_Hat_OpenStack_Platform-14-Networking_Guide-en-US/images/1c872c1cb32da36ce85f48656ad056d8/OpenStack_Networking-Guide_471659_0518_LBaaS-Topology.png)

![Example architecture](https://docs.infomaniak.cloud/pictures/0020.load_balancer_1.png)

#### Deploy a basic HTTP load balancer
While this is technically the simplest complete load balancing solution that can be deployed, it is recommended to deploy HTTP load balancers with a health monitor to ensure back-end member availability. 

Scenario description:

- Back-end servers 192.0.2.10 and 192.0.2.11 on subnet private-subnet have been configured with an HTTP application on TCP port 80.
- Subnet public-subnet is a shared external subnet created by the cloud operator which is reachable from the internet.
- We want to configure a basic load balancer that is accessible from the internet, which distributes web requests to the back-end servers.

Solution:

- Create load balancer lb1 on subnet public-subnet.
- Create listener listener1.
- Create pool pool1 as listener1’s default pool.
- Add members 192.0.2.10 and 192.0.2.11 on private-subnet to pool1.

CLI commands:

```
    openstack loadbalancer create --name lb1 --vip-subnet-id public-subnet
    # Re-run the following until lb1 shows ACTIVE and ONLINE statuses:
    openstack loadbalancer show lb1
    openstack loadbalancer listener create --name listener1 --protocol HTTP --protocol-port 80 lb1
    openstack loadbalancer pool create --name pool1 --lb-algorithm ROUND_ROBIN --listener listener1 --protocol HTTP
    openstack loadbalancer member create --subnet-id private-subnet --address 192.0.2.10 --protocol-port 80 pool1
    openstack loadbalancer member create --subnet-id private-subnet --address 192.0.2.11 --protocol-port 80 pool1
```

#### Deploy a basic HTTP load balancer with a health monitor
This is the simplest recommended load balancing solution for HTTP applications. This solution is appropriate for operators with provider networks that are not compatible with Neutron floating-ip functionality (such as IPv6 networks). However, if you need to retain control of the external IP through which a load balancer is accessible, even if the load balancer needs to be destroyed or recreated, it may be more appropriate to deploy your basic load balancer using a floating IP. 

Scenario description:
- Back-end servers 192.0.2.10 and 192.0.2.11 on subnet private-subnet have been configured with an HTTP application on TCP port 80.
- These back-end servers have been configured with a health check at the URL path “/healthcheck”. See Configuration arguments for HTTP health monitors below.
- Subnet public-subnet is a shared external subnet created by the cloud operator which is reachable from the internet.
- We want to configure a basic load balancer that is accessible from the internet, which distributes web requests to the back-end servers, and which checks the “/healthcheck” path to ensure back-end member health.

Solution:
- Create load balancer lb1 on subnet public-subnet.
- Create listener listener1.
- Create pool pool1 as listener1’s default pool.
- Create a health monitor on pool1 which tests the “/healthcheck” path.
- Add members 192.0.2.10 and 192.0.2.11 on private-subnet to pool1.

CLI commands:

```
openstack loadbalancer create --name lb1 --vip-subnet-id public-subnet
# Re-run the following until lb1 shows ACTIVE and ONLINE statuses:
openstack loadbalancer show lb1
openstack loadbalancer listener create --name listener1 --protocol HTTP --protocol-port 80 lb1
openstack loadbalancer pool create --name pool1 --lb-algorithm ROUND_ROBIN --listener listener1 --protocol HTTP
openstack loadbalancer healthmonitor create --delay 5 --max-retries 4 --timeout 10 --type HTTP --url-path /healthcheck pool1
openstack loadbalancer member create --subnet-id private-subnet --address 192.0.2.10 --protocol-port 80 pool1
openstack loadbalancer member create --subnet-id private-subnet --address 192.0.2.11 --protocol-port 80 pool1
```

#### Deploy a basic HTTP load balancer using a floating IP
It can be beneficial to use a floating IP when setting up a load balancer’s VIP in order to ensure you retain control of the IP that gets assigned as the floating IP in case the load balancer needs to be destroyed, moved, or recreated.

Note that this is not possible to do with IPv6 load balancers as floating IPs do not work with IPv6.

Scenario description:

- Back-end servers 192.0.2.10 and 192.0.2.11 on subnet private-subnet have been configured with an HTTP application on TCP port 80.
- These back-end servers have been configured with a health check at the URL path “/healthcheck”. See Configuration arguments for HTTP health monitors below.
- Neutron network public is a shared external network created by the cloud operator which is reachable from the internet.
- We want to configure a basic load balancer that is accessible from the internet, which distributes web requests to the back-end servers, and which checks the “/healthcheck” path to ensure back-end member health. Further, we want to do this using a floating IP.

Solution:

- Create load balancer lb1 on subnet private-subnet.
- Create listener listener1.
- Create pool pool1 as listener1’s default pool.
- Create a health monitor on pool1 which tests the “/healthcheck” path.
- Add members 192.0.2.10 and 192.0.2.11 on private-subnet to pool1.
- Create a floating IP address on public-subnet.
- Associate this floating IP with the lb1’s VIP port.

CLI commands:

```
openstack loadbalancer create --name lb1 --vip-subnet-id private-subnet
# Re-run the following until lb1 shows ACTIVE and ONLINE statuses:
openstack loadbalancer show lb1
openstack loadbalancer listener create --name listener1 --protocol HTTP --protocol-port 80 lb1
openstack loadbalancer pool create --name pool1 --lb-algorithm ROUND_ROBIN --listener listener1 --protocol HTTP
openstack loadbalancer healthmonitor create --delay 5 --max-retries 4 --timeout 10 --type HTTP --url-path /healthcheck pool1
openstack loadbalancer member create --subnet-id private-subnet --address 192.0.2.10 --protocol-port 80 pool1
openstack loadbalancer member create --subnet-id private-subnet --address 192.0.2.11 --protocol-port 80 pool1
openstack floating ip create public
# The following IDs should be visible in the output of previous commands
openstack floating ip set --port <load_balancer_vip_port_id> <floating_ip_id>
```

#### Deploy a basic HTTP load balancer using a floating IP
It can be beneficial to use a floating IP when setting up a load balancer’s VIP in order to ensure you retain control of the IP that gets assigned as the floating IP in case the load balancer needs to be destroyed, moved, or recreated.

Note that this is not possible to do with IPv6 load balancers as floating IPs do not work with IPv6.

Scenario description:

- Back-end servers 192.0.2.10 and 192.0.2.11 on subnet private-subnet have been configured with an HTTP application on TCP port 80.
- These back-end servers have been configured with a health check at the URL path “/healthcheck”. See Configuration arguments for HTTP health monitors below.
- Neutron network public is a shared external network created by the cloud operator which is reachable from the internet.
- We want to configure a basic load balancer that is accessible from the internet, which distributes web requests to the back-end servers, and which checks the “/healthcheck” path to ensure back-end member health. Further, we want to do this using a floating IP.

Solution:

- Create load balancer lb1 on subnet private-subnet.
- Create listener listener1.
- Create pool pool1 as listener1’s default pool.
- Create a health monitor on pool1 which tests the “/healthcheck” path.
- Add members 192.0.2.10 and 192.0.2.11 on private-subnet to pool1.
- Create a floating IP address on public-subnet.
- Associate this floating IP with the lb1’s VIP port.

CLI commands:

```
openstack loadbalancer create --name lb1 --vip-subnet-id private-subnet
# Re-run the following until lb1 shows ACTIVE and ONLINE statuses:
openstack loadbalancer show lb1
openstack loadbalancer listener create --name listener1 --protocol HTTP --protocol-port 80 lb1
openstack loadbalancer pool create --name pool1 --lb-algorithm ROUND_ROBIN --listener listener1 --protocol HTTP
openstack loadbalancer healthmonitor create --delay 5 --max-retries 4 --timeout 10 --type HTTP --url-path /healthcheck pool1
openstack loadbalancer member create --subnet-id private-subnet --address 192.0.2.10 --protocol-port 80 pool1
openstack loadbalancer member create --subnet-id private-subnet --address 192.0.2.11 --protocol-port 80 pool1
openstack floating ip create public
# The following IDs should be visible in the output of previous commands
openstack floating ip set --port <load_balancer_vip_port_id> <floating_ip_id>
```

## Certificates, keys and configuration 

We have seen above that the control plane components of Octavia use a REST API exposed by the agent running on each amphora to make changes to the configuration of the HAProxy instances. Obviously, this connection needs to be secured. For that purpose, Octavia uses TLS certificates.

First, there is a client certificate. This certificate will be used by the control plane to authenticate itself when connecting to the agent. The client certificate and the corresponding key need to be created during installation. As the CA certificate used to sign the client certificate needs to be present on every amphora (so that the agent can use it to verify incoming requests), this certificate needs to be known to Octavia as well, and Octavia will distribute it to each newly created amphora.

Second, each agent of course needs a server certificate. These certificates are unique to each agent and are created dynamically at run time by a certificate generator built into the Octavia control plane. During the installation, we only have to provide a CA certificate and a corresponding private key which Octavia will then use to issue the server certificates. The following diagram summarizes the involved certificates and key.

In addition, Octavia can place an SSH key on each amphora to allow us to SSH into an amphora in case there are any issues with it. And finally, a short string is used as a secret to encrypt the health status messages. Thus, during the installation, we have to create: 
- A root CA certificate that will be placed on each amphora
- A client certificate signed by this root CA and a corresponding client key
- An additional root CA certificate that Octavia will use to create the server certificates and a corresponding key
- An SSH key pair
- A secret for the encryption of the health messages

### Two-way TLS Authentication 

The Octavia controller processes communicate with the Amphora over a TLS connection much like an HTTPS connection to a website. However, Octavia validates that both sides are trusted by doing a two-way TLS authentication.

Phase One
When a controller process, such as the Octavia worker process, connects to an Amphora, the Amphora will present its server certificate to the controller. The controller will then validate it against the server Certificate Authority (CA) certificate stored on the controller. If the presented certificate is validated against the server CA certificate, the connection goes into phase two of the two-way TLS authentication.

Phase Two
Once phase one is complete, the controller will present its client certificate to the Amphora. The Amphora will then validate the certificate against the client CA certificate stored inside the Amphora. If this certificate is successfully validated, the rest of the TLS handshake will continue to establish the secure communication channel between the controller and the Amphora.

Certificate Lifecycles
The server certificates are uniquely generated for each amphora by the controller using the server certificate authority certificates and keys. These server certificates are automatically rotated by the Octavia housekeeping controller process as they near expiration.

The client certificates are used for the Octavia controller processes. These are managed by the operator and due to their use on the control plane of the cloud, typically have a long lifetime.

## Load balancer network 

A load balancer network is used to distribute incoming network traffic across multiple servers or instances in order to improve performance, scalability, and availability of applications or services. Here are the main purposes and benefits of using a load balancer network:

- Load Distribution: A load balancer evenly distributes incoming network traffic across multiple backend servers or instances. This helps distribute the workload and prevents any single server from becoming overloaded, ensuring optimal performance and resource utilization.

- Scalability: Load balancers enable horizontal scalability by allowing you to add or remove backend servers or instances dynamically. As your application or service demand increases, you can easily scale by adding more servers or instances behind the load balancer to handle the additional traffic.

- High Availability: Load balancers enhance the availability of your application or service by providing redundancy. If one backend server or instance fails, the load balancer automatically redirects traffic to the remaining healthy servers, ensuring continuity and minimizing downtime.

- Health Monitoring and Failover: Load balancers can periodically check the health and availability of backend servers or instances. If a server or instance fails the health check, the load balancer can automatically remove it from the pool of available servers, preventing it from receiving traffic until it recovers or is replaced. This helps ensure that only healthy servers handle incoming requests.

- SSL/TLS Termination: Load balancers can offload SSL/TLS encryption and decryption, relieving the backend servers from the computational burden. The load balancer handles the SSL/TLS handshake and encryption, forwarding the decrypted traffic to the backend servers, and encrypting the responses before sending them back to clients. This improves performance and reduces the resource requirements on the backend servers.

- Session Persistence: Load balancers can provide session persistence, also known as sticky sessions or session affinity. With session persistence, subsequent requests from a client are routed to the same backend server that initially served the client's request. This is useful for applications that require maintaining session state or have specific requirements for handling client sessions.

Overall, a load balancer network plays a crucial role in optimizing the performance, scalability, and availability of applications or services by efficiently distributing network traffic among multiple servers or instances. It helps ensure a smooth and responsive user experience while maintaining high availability and facilitating easy scalability.

## Keywords 

- Amphora: Virtual machine, container, dedicated hardware, appliance or device that actually performs the task of load balancing in the Octavia system. More specifically, an amphora takes requests from clients on the front-end and distributes these to back-end systems. Amphorae communicate with their controllers over the LB Network through a driver interface on the controller.

- VIP (Virtual IP address): Single service IP address which is associated with a load balancer. This is similar to what is described here: http://en.wikipedia.org/wiki/Virtual_IP_address. In a highly available load balancing topology in Octavia, the VIP might be assigned to several amphorae, and a layer-2 protocol like CARP, VRRP, or HSRP (or something unique to the networking infrastructure) might be used to maintain its availability. In layer-3 (routed) topologies, the VIP address might be assigned to an upstream networking device which routes packets to amphorae, which then load balance requests to back-end members.

- TLS Termination (Transport Layer Security Termination): Type of load balancing protocol where HTTPS sessions are terminated (decrypted) on the amphora as opposed to encrypted packets being forwarded on to back-end servers without being decrypted on the amphora. Also known as SSL termination. The main advantages to this type of load balancing are that the payload can be read and / or manipulated by the amphora, and that the expensive tasks of handling the encryption are off-loaded from the back-end servers. This is particularly useful if layer 7 switching is employed in the same listener configuration. 

- LB Network (Load Balancer Network): The network over which the controller(s) and amphorae communicate. The LB network itself will usually be a nova or neutron network to which both the controller and amphorae have access, but is not associated with any one tenant. The LB Network is generally also not part of the undercloud and should not be directly exposed to any OpenStack core components other than the Octavia Controller.

- HAProxy: Load balancing software used in the reference implementation for Octavia. (See http://www.haproxy.org/ ). HAProxy processes run on amphorae and actually accomplish the task of delivering the load balancing service.

- Listener: The listening endpoint,for example HTTP, of a load balanced service. A listener might refer to several pools (and switch between them using layer 7 rules).

- Pool: A group of members that handle client requests from the load balancer (amphora). A pool is associated with only one listener.

- Member: Compute instances that serve traffic behind the load balancer (amphora) in a pool.

- Undercloud: the deployment cloud that contain the necessary OpenStack components to deploy and manage an Overcloud also called workload cloud or deployed cloud. 

- Overcloud: the deployed solution and can be used for production, staging, or testing, etc.



## Resources 
https://docs.openstack.org/octavia/latest/reference/introduction.html
https://docs.openstack.org/octavia/latest/reference/glossary.html
https://docs.openstack.org/octavia/latest/user/guides/basic-cookbook.html
https://docs.openstack.org/octavia/latest/admin/guides/certificates.html
https://leftasexercise.com/2020/05/01/openstack-octavia-architecture-and-installation/