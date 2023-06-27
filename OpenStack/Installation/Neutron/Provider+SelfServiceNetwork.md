
"Self-service networking" allows users to create their own virtual networks, subnets, routers, etc., where as the "provider networking" does not allow users to create new virtual networking components and allow them to use only the ones that are predefined by the provider.

Provider-networks are normally tied to your already-existent switching architecture, and can be used as "external-network" for OpenStack. The FIP's (floating IP's) are taken from that "external network". Normally, FLAT or VLAN based networks are used for provider-networks.

The self-service are, like the answer above, created internally by the tenant/project. Those self-service are tunneled type networks (using both GRE and/or VXLAN), and "tied" in a SNAT - source NAT / DNAT - destination NAT scheme to external networks (provider-type) using a OpenStack router.

The OpenStack router has a "leg" attached to the external network, and another leg into the self-service, internal network.

Then, depending of your environment and OpenStack release (specially from Kilo to Mitaka) you can have multiple external networks, and combine different types of tunneled self-service ones (VXLAN or GRE). As a matter of facts, you can both assign FIP to your instances, or directly assign IP's from the provider-type net's (permissions provided of course) to your instances.



Key terms: 
- Tunneling: https://traefik.io/glossary/network-tunneling/ 
- VLAN: https://www.makeuseof.com/what-is-a-vlan-and-how-does-it-work/ 
- GRE: Encapsulating packets within other packets is called "tunneling." GRE tunnels are usually configured between two routers, with each router acting like one end of the tunnel. The routers are set up to send and receive GRE packets directly to each other. Any routers in between those two routers will not open the encapsulated packets; they only reference the headers surrounding the encapsulated packets in order to forward them. https://www.cloudflare.com/en-gb/learning/network-layer/what-is-gre-tunneling/ 
- VXLAN (Virtual Extensible LAN): DC virtualization https://viblo.asia/p/vxlan-cong-nghe-ao-hoa-dc-1Je5EQLL5nL 

Packet sending to the Internet: https://docs.openstack.org/neutron/pike/_images/deploy-lb-provider-flowns1.png

Packet sending between instances on the same network: https://docs.openstack.org/neutron/pike/_images/deploy-lb-provider-flowew1.png

Packet sending between instances on different networks: https://docs.openstack.org/neutron/pike/_images/deploy-lb-provider-flowew2.png


References:
- https://stackoverflow.com/questions/36747239/what-is-the-difference-between-provider-network-and-self-service-network-in-open#:~:text=%22self%2Dservice%20networking%22%20allows,are%20predefined%20by%20the%20provider.
- https://leftasexercise.com/2020/02/21/openstack-neutron-architecture-and-overview/
- https://docs.openstack.org/neutron/pike/admin/deploy-lb-provider.html