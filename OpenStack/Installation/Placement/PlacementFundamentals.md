## Placement 

This is a REST API stack and data model used to track resource provider inventories and usages, along with different classes of resources. For example, a resource provider can be a compute node, a shared storage pool, or an IP allocation pool. The placement service tracks the inventory and usage of each provider. For example, an instance created on a compute node may be a consumer of resources such as RAM and CPU from a compute node resource provider, disk from an external shared storage pool resource provider and IP addresses from an external IP pool resource provider.

The types of resources consumed are tracked as classes. The service provides a set of standard resource classes (for example DISK_GB, MEMORY_MB, and VCPU) and provides the ability to define custom resource classes as needed.

Each resource provider may also have a set of traits which describe qualitative aspects of the resource provider. Traits describe an aspect of a resource provider that cannot itself be consumed but a workload may wish to specify. For example, available disk may be solid state drives (SSD).

### Tracking Resources
The placement service enables other projects to track their own resources. Those projects can register/delete their own resources to/from placement via the placement HTTP API.

The placement service originated in the Nova project. As a result much of the functionality in placement was driven by nova’s requirements.

Deploy the API service: 
Placement provides a placement-api WSGI script for running the service with Apache, nginx or other WSGI-capable web servers. Depending on what packaging solution is used to deploy OpenStack, the WSGI script may be in /usr/bin or /usr/local/bin.

**placement-api**, as a standard WSGI script, provides a module level application attribute that most WSGI servers expect to find. This means it is possible to run it with lots of different servers, providing flexibility in the face of different deployment scenarios.


Key terms: 
- Resource class: type of resources that Placement manages, like IP addresses, vCPUs, disk space or memory 
- Inventory: each provide offers a certain set of resource classes 
- Allocation: represents a usage of resources by a specific consumer

### Resources 
(https://leftasexercise.files.wordpress.com/2019/11/placementdatamodel.png)[https://leftasexercise.files.wordpress.com/2019/11/placementdatamodel.png]
(https://docs.openstack.org/placement/victoria/install/)[https://docs.openstack.org/placement/victoria/install/]