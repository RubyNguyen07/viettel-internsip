## Neutron 

OpenStack Networking (neutron) allows you to create and attach interface devices managed by other OpenStack services to networks. Plug-ins can be implemented to accommodate different networking equipment and software, providing flexibility to OpenStack architecture and deployment.

It includes the following components: 
- neutron-server: Accepts and routes API requests to the appropriate OpenStack Networking plug-in for action.
- OpenStack Networking plug-ins and agents: Plug and unplug ports, create networks or subnets, and provide IP addressing. These plug-ins and agents differ depending on the vendor and technologies used in the particular cloud. OpenStack Networking ships with plug-ins and agents for Cisco virtual and physical switches, NEC OpenFlow products, Open vSwitch, Linux bridging, and the VMware NSX product. The common agents are L3 (layer 3), DHCP (dynamic host IP addressing), and a plug-in agent.
- Messaging queue: Used by most OpenStack Networking installations to route information between the neutron-server and various agents. Also acts as a database to store networking state for particular plug-ins.

OpenStack Networking mainly interacts with OpenStack Compute to provide networks and connectivity for its instances.

OpenStack Networking is entirely standalone and can be deployed to a dedicated host. If your deployment uses a controller host to run centralized Compute components, you can deploy the Networking server to that specific host instead.

In the first diagram, the provider network relies on physical network infrastructure for layer-3 (routing) services. 
[Diagram for provider network](https://docs.openstack.org/neutron/victoria/_images/network1-services.png)
[Diagram for self-service network](https://docs.openstack.org/neutron/victoria/_images/network2-services.png)

### Network types 

Regardless of the network type used, Neutron networks can be external or internal. External networks are networks that allow for connectivity to networks outside of the OpenStack deployment, like the Internet, whereas internal networks are isolated. Technically, Neutron does not really know whether a network has connectivity to the outside world, therefore “external network” is essentially a flag attached to a network which becomes relevant when we discuss IP routers in a later post.


### Some terms  

Networks in Neutron are layer 2 networks, and if two compute instances are assigned to the same virtual network, they are connected to an actual virtual Ethernet segment and can reach each other on the Ethernet level.

What technologies do we have available to realize this? In a first step, let us focus on connecting two different virtual machines running on the same host. So assume that we are given two virtual machines, call them VM1 and VM2, on the same physical compute node. Our hypervisor will attach a virtual interface (VIF) to each of these virtual machines. In a physical network, you would simply connect these two interfaces to ports of a switch to connect the instances. In our case, we can use a virtual switch / bridge to achieve this.

OpenStack is able to leverage several bridging technologies. First, OpenStack can of course use the Linux bridge driver to build and configure virtual switches. In addition, Neutron comes with a driver that uses Open vSwitch (OVS). Throughout this series, I will focus on the use of OVS as a virtual switch.

So, to connect the VMs running on the same host, Neutron could use (and it actually does) an OVS bridge to which the virtual machine networking interfaces are attached. This bridge is called the integration bridge. We will see later that, as in a typical physical network, this bridge is also connected to a DHCP agent, routers and so forth.

https://leftasexercise.files.wordpress.com/2019/11/neutronnetworkingstepi.png

But even for the simple case of VMs on the same host, we are not yet done. In reality, to operate a cloud at scale, you will need some approach to isolate networks. If, for instance, the two VMs belong to different tenants, you do not want them to be on the same network. To do this, Neutron uses VLANs. So the ports connecting the integration bridge to the individual VMs are tagged, and there is one VLAN for each Neutron network.

This networking type is called a local network in Neutron. It is possible to set up Neutron to only use this type of network, but in reality, this is of course not really useful. Instead, we need to move on and connect the VMs that are attached to the same network on different hosts. To do this, we will have to use some virtual networking technology to connect the integration bridges on the different hosts.

First, we could simply connect each integration bridge to a physical network device which in turn is connected to the physical network. With this setup, called a flat network in Neutron, all virtual machines are effectively connected to the same Ethernet segment. Consequently, there can only be one flat network per deployment.

The second option we have is to use VLANs to partition the physical network according to the virtual networks that we wish to establish. In this approach, Neutron would assign a global VLAN ID to each virtual network (which in general is different from the VLAN ID used on the integration bridge) and tag the traffic within each virtual network with the corresponding VLAN ID before handing it over to the physical network infrastructure.

Finally, we could use tunnels to connect the integration bridges across the hosts. Neutron supports the most commonly used tunneling protocols (VXLAN, GRE, Geneve).

### Architecture 

https://leftasexercise.files.wordpress.com/2019/11/neutronarchitecture.png

Now let us take a closer look at the ML2 plugin. This plugin again utilizes pluggable modules called drivers. There are two types of drivers. First, there are type drivers which provide functionality for a specific network type, like a flat network, a VXLAN network, a VLAN network and so forth. Second, there are mechanism drivers that contain the logic specific to an implementation, like OVS or Linux bridging. Typically, the mechanism driver will in turn communicate with an L2 agent like the OVS agent running on the compute nodes.

### Concepts 
OpenStack Networking (neutron) manages all networking facets for the Virtual Networking Infrastructure (VNI) and the access layer aspects of the Physical Networking Infrastructure (PNI) in your OpenStack environment. OpenStack Networking enables projects to create advanced virtual network topologies which may include services such as a firewall, and a virtual private network (VPN).

Networking provides networks, subnets, and routers as object abstractions. Each abstraction has functionality that mimics its physical counterpart: networks contain subnets, and routers route traffic between different subnets and networks.

Any given Networking set up has at least one external network. Unlike the other networks, the external network is not merely a virtually defined network. Instead, it represents a view into a slice of the physical, external network accessible outside the OpenStack installation. IP addresses on the external network are accessible by anybody physically on the outside network.

In addition to external networks, any Networking set up has one or more internal networks. These software-defined networks connect directly to the VMs. Only the VMs on any given internal network, or those on subnets connected through interfaces to a similar router, can access VMs connected to that network directly.

For the outside network to access VMs, and vice versa, routers between the networks are needed. Each router has one gateway that is connected to an external network and one or more interfaces connected to internal networks. Like a physical router, subnets can access machines on other subnets that are connected to the same router, and machines can access the outside network through the gateway for the router.

Additionally, you can allocate IP addresses on external networks to ports on the internal network. Whenever something is connected to a subnet, that connection is called a port. You can associate external network IP addresses with ports to VMs. This way, entities on the outside network can access VMs.

Each plug-in that Networking uses has its own concepts. While not vital to operating the VNI and OpenStack environment, understanding these concepts can help you set up Networking. All Networking installations use a core plug-in and a security group plug-in (or just the No-Op security group plug-in). Additionally, Firewall-as-a-Service (FWaaS) is available.

### Security group

Security groups are sets of IP filter rules that are applied to all project instances, which define networking access to the instance. Group rules are project specific; project members can edit the default rules for their group and add new rule sets.

All projects have a default security group which is applied to any instance that has no other defined security group. Unless you change the default, this security group denies all incoming traffic and allows only outgoing traffic to your instance.

Rules are allow rules as the default is deny. 

#### Security port 

The OpenStack platform, specifically Neutron (the networking component), uses the concepts of “ports” in order to connect the various cloud instances to different networks and the corresponding virtual networking devices like Neutron routers, firewalls etc.

OpenStack allows us to manage security on the individual port level in an environment. By default, the following rules apply:

- All incoming and outgoing traffic is blocked for ports connected to virtual machine instances. (Unless a ‘Security Group’ has been applied.)
- Only traffic originating from the IP / MAC address pair known to OpenStack for a particular port, will be allowed on the network.
- Pass through and promiscuous mode will be blocked.

### Quality of Service 

QoS is defined as the ability to guarantee certain network requirements like bandwidth, latency, jitter, and reliability in order to satisfy a Service Level Agreement (SLA) between an application provider and end users.

Network devices such as switches and routers can mark traffic so that it is handled with a higher priority to fulfill the QoS conditions agreed under the SLA. In other cases, certain network traffic such as Voice over IP (VoIP) and video streaming needs to be transmitted with minimal bandwidth constraints. On a system without network QoS management, all traffic will be transmitted in a “best-effort” manner making it impossible to guarantee service delivery to customers.

QoS is an advanced service plug-in. QoS is decoupled from the rest of the OpenStack Networking code on multiple levels and it is available through the ml2 extension driver.

QoS supported rule types are now available as VALID_RULE_TYPES in QoS rule types:
- bandwidth_limit: Bandwidth limitations on networks, ports or floating IPs.
- packet_rate_limit: Packet rate limitations on certain types of traffic.
- dscp_marking: Marking network traffic with a DSCP value.
- minimum_bandwidth: Minimum bandwidth constraints on certain types of traffic.
- minimum_packet_rate: Minimum packet rate constraints on certain types of traffic.

### Key terms: 
iptables: https://linux.die.net/man/8/iptables 

iptables -t nat -A POSTROUTING -j MASQUERADE: MASQUERADE does what the name suggests: It hides everything “behind” the host. You’d do that to supply Internet to multiple hosts when you only have one uplink IP address. This tech is used on most consumer-grade Internet access routers, dubbed “NAT”. When host A contacts server S via MASQUERADEing router R, the server won’t be able to see the connection originates from host A. Instead, to the server it looks like it’s communicating with router R. Router R however knows this connection was originally from host A and will forward messages accordingly. Host A knows it connected to server S and that server S sent the response.

Port: A port is a connection point for attaching a single device, such as the NIC of a server, to a network. The port also describes the associated network configuration, such as the MAC and IP addresses to be used on that port.

ML2 plugin: The Modular Layer 2 (ml2) plugin is a framework allowing OpenStack Networking to simultaneously utilize the variety of layer 2 networking technologies found in complex real-world data centers.

Type driver: Each available network type is managed by an ml2 TypeDriver. TypeDrivers maintain any needed type-specific network state, and perform provider network validation and tenant network allocation. The ml2 plugin currently includes drivers for the local, flat, vlan, gre and vxlan network types.

Mechanism driver: Each networking mechanism is managed by an ml2 MechanismDriver. The MechanismDriver is responsible for taking the information established by the TypeDriver and ensuring that it is properly applied given the specific networking mechanisms that have been enabled.

L2population: L2 population is mechanism driver for ML2 plugin which tends to leverage the implementation of overlay networks. By populating the forwarding tables of virtual switches (LinuxBridge or OVS), l2population mech driver will decrease broadcast traffics inside the physical networks fabric while using overlays networks (VXLan, GRE).

Security group: 

DHCP service: qdhcp namespaces and dnsmasq service. 

OpenvSwitch (OVS): is an open-source implementation of a distributed virtual multilayer switch.



### Resources 
https://docs.openstack.org/neutron/victoria/install/concepts.html
https://docs.openstack.org/neutron/victoria/install/common/get-started-networking.html
https://docs.openstack.org/nova/train/admin/security-groups.html
https://superuser.openinfra.dev/articles/managing-port-level-security-openstack/
https://docs.openstack.org/neutron/latest/admin/config-qos.html