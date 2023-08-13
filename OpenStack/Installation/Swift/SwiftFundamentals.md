## Swift 

The OpenStack Object Storage is a multi-tenant object storage system. It is highly scalable and can manage large amounts of unstructured data at low cost through a RESTful HTTP API.

It includes the following components:
- Proxy servers (swift-proxy-server): Accepts OpenStack Object Storage API and raw HTTP requests to upload files, modify metadata, and create containers. It also serves file or container listings to web browsers. To improve performance, the proxy server can use an optional cache that is usually deployed with memcache.

- Account servers (swift-account-server): Manages accounts defined with Object Storage.

- Container servers (swift-container-server): Manages the mapping of containers or folders, within Object Storage.

- Object servers (swift-object-server): Manages actual objects, such as files, on the storage nodes.

- Various periodic processes: Performs housekeeping tasks on the large data store. The replication services ensure consistency and availability through the cluster. Other periodic processes include auditors, updaters, and reapers.

- WSGI middleware: Handles authentication and is usually OpenStack Identity.

- swift client: Enables users to submit commands to the REST API through a command-line client authorized as either a admin user, reseller user, or swift user.

- swift-init: Script that initializes the building of the ring file, takes daemon names as parameter and offers commands. Documented in https://docs.openstack.org/swift/latest/admin_guide.html#managing-services.

- swift-recon: A cli tool used to retrieve various metrics and telemetry information about a cluster that has been collected by the swift-recon middleware.

- swift-ring-builder: Storage ring build and rebalance utility. Documented in https://docs.openstack.org/swift/latest/admin_guide.html#managing-the-rings.


### Containers vs Rings 

- Container:

+ A container in OpenStack Swift is a logical storage container that holds a collection of objects. It acts as a top-level namespace for organizing objects.
+ Containers are similar to directories or folders in a traditional file system hierarchy.
+ Containers can be created, deleted, listed, and managed through the Swift API or command-line tools.
+ Each container has its own unique name and can store a large number of objects.

- Ring:

+ The ring in OpenStack Swift refers to the Ring Builder, which is a part of the Swift infrastructure responsible for data placement and distribution across storage nodes.
+ The ring contains information about the storage devices, their locations, and the partitions to which they are assigned.
+ The Ring Builder generates a consistent hash ring that maps objects to specific partitions and determines which storage nodes are responsible for storing those partitions.
+ The ring allows Swift to distribute data across multiple storage nodes in a scalable and fault-tolerant manner.
+ When a client sends a request to Swift to access an object, the ring is consulted to determine which storage node holds the data and retrieves it from there.

In summary, containers in OpenStack Swift provide a way to organize and group objects logically, while the ring is a critical component of Swift's distributed storage architecture that determines the placement and retrieval of objects across multiple storage nodes.

### Container

In OpenStack Swift, a container is not located inside a specific storage node. Instead, a container is a logical construct that exists within the Swift cluster as a whole.

The Swift cluster consists of multiple storage nodes, each responsible for storing a portion of the data and participating in data replication. The data in Swift is distributed across these storage nodes to achieve scalability, fault tolerance, and high availability.

When a client interacts with Swift to create or access a container, the request is handled by the proxy server component of Swift. The proxy server determines the appropriate storage nodes for the container based on the ring data and forwards the request accordingly.

The actual data associated with the container, which includes the objects stored within it, is distributed across the storage nodes assigned to handle the specific partitions of the container's data. This distribution is managed by the ring and handled transparently by Swift without the client needing to know the specific location of the data within the storage nodes.

In summary, containers are logical constructs within the Swift cluster, while the data associated with the container is distributed across multiple storage nodes in the cluster. This distributed architecture enables Swift to achieve scalability, fault tolerance, and efficient data management.

#### Purpose and use 

- Organization and Hierarchical Structure: Containers provide a way to organize objects hierarchically, similar to directories or folders in a traditional file system. Objects within a container can be logically grouped based on their purpose, content type, project, user, or any other relevant criteria. Containers can have sub-containers, allowing for the creation of a hierarchical structure to organize objects in a more granular manner.

- Access Control and Permissions: Containers have their own access control mechanisms, allowing administrators to define access policies and permissions for different users or groups. Access controls can be set at the container level, granting read, write, or delete permissions to specific users or accounts. This enables fine-grained access control and helps enforce security and privacy requirements.

- Metadata and Policies: Containers in Swift support metadata, which are key-value pairs associated with the container. Metadata can be used to provide additional information about the container, such as the purpose, creation date, owner, or any custom attributes. Policies can be applied at the container level to enforce specific behaviors or rules, such as data retention policies or replication settings.

- Scalability and Performance: Containers play a role in achieving scalability and performance in Swift. Containers act as a unit of storage and allow for efficient distribution of objects across multiple storage nodes in the Swift cluster. Objects within a container can be distributed and replicated across different drives and servers, ensuring redundancy and high availability.

- Object Listing and Management: Containers provide the capability to list, search, and manage the objects stored within them. Users can query a container to retrieve a list of objects, filter based on metadata, or perform bulk operations on multiple objects within the container. Swift APIs and client libraries provide various methods to interact with containers, allowing users to create, delete, update, or retrieve information about containers and their contents.


### The Rings 

![What is Ring](https://www.itweet.cn/screenshots/openstack-swift.png)

A ring represents a mapping between the names of entities stored on disk and their physical location. There are separate rings for accounts, containers, and one object ring per storage policy. When other components need to perform any operation on an object, container, or account, they need to interact with the appropriate ring to determine its location in the cluster.

The rings determine where data should reside in the cluster. There is a separate ring for account databases, container databases, and individual object storage policies but each ring works in the same way. These rings are externally managed. The server processes themselves do not modify the rings; they are instead given new rings modified by other tools.

The ring uses a configurable number of bits from the MD5 hash of an item’s path as a partition index that designates the device(s) on which that item should be stored. The number of bits kept from the hash is known as the partition power, and 2 to the partition power indicates the partition count. Partitioning the full MD5 hash ring allows the cluster components to process resources in batches. This ends up either more efficient or at least less complex than working with each item separately or the entire cluster all at once.

Another configurable value is the replica count, which indicates how many devices to assign for each partition in the ring. By having multiple devices responsible for each partition, the cluster can recover from drive or network failures.

Devices are added to the ring to describe the capacity available for partition replica assignments. Devices are placed into failure domains consisting of region, zone, and server. Regions can be used to describe geographical systems characterized by lower bandwidth or higher latency between machines in different regions. Many rings will consist of only a single region. Zones can be used to group devices based on physical locations, power separations, network separations, or any other attribute that would lessen multiple replicas being unavailable at the same time.

Devices are given a weight which describes the relative storage capacity contributed by the device in comparison to other devices.

When building a ring, replicas for each partition will be assigned to devices according to the devices’ weights. Additionally, each replica of a partition will preferentially be assigned to a device whose failure domain does not already have a replica for that partition. Only a single replica of a partition may be assigned to each device - you must have at least as many devices as replicas.


![Demonstration](https://avcourt.github.io/tiny-cluster/assets/openstack/hash_ring.png)
![Region and Zone](https://platform.swiftstack.com/docs/_images/cluster-regions.png)

#### Ring Builder 

The rings are built and managed manually by a utility called the ring-builder. The ring-builder assigns partitions to devices and writes an optimized structure to a gzipped, serialized file on disk for shipping out to the servers. The server processes check the modification time of the file occasionally and reload their in-memory copies of the ring structure as needed. Because of how the ring-builder manages changes to the ring, using a slightly older ring usually just means that for a subset of the partitions the device for one of the replicas will be incorrect, which can be easily worked around.

The ring-builder also keeps a separate builder file which includes the ring information as well as additional data required to build future rings. It is very important to keep multiple backup copies of these builder files. One option is to copy the builder files out to every server while copying the ring files themselves. Another is to upload the builder files into the cluster itself. Complete loss of a builder file will mean creating a new ring from scratch, nearly all partitions will end up assigned to different devices, and therefore nearly all data stored will have to be replicated to new locations. So, recovery from a builder file loss is possible, but data will definitely be unreachable for an extended time.

#### Ring Data Structure

The ring data structure consists of three top level fields: a list of devices in the cluster, a list of lists of device ids indicating partition to device assignments, and an integer indicating the number of bits to shift an MD5 hash to calculate the partition for the hash. The list of devices is known internally to the Ring class as devs.

### Storage Policies 

Storage Policies provide a way for object storage providers to differentiate service levels, features and behaviors of a Swift deployment. Each Storage Policy configured in Swift is exposed to the client via an abstract name. Each device in the system is assigned to one or more Storage Policies. This is accomplished through the use of multiple object rings, where each Storage Policy has an independent object ring, which may include a subset of hardware implementing a particular differentiation.

For example, one might have the default policy with 3x replication, and create a second policy which, when applied to new containers only uses 2x replication. Another might add SSDs to a set of storage nodes and create a performance tier storage policy for certain containers to have their objects stored there. Yet another might be the use of Erasure Coding to define a cold-storage tier.

This mapping is then exposed on a per-container basis, where each container can be assigned a specific storage policy when it is created, which remains in effect for the lifetime of the container. Applications require minimal awareness of storage policies to use them; once a container has been created with a specific policy, all objects stored in it will be done so in accordance with that policy.

The Storage Policies feature is implemented throughout the entire code base so it is an important concept in understanding Swift architecture.

### In general 

A client uses the REST API to make a HTTP request to PUT an object into an existing container. The cluster receives the request. First, the system must figure out where the data is going to go. To do this, the account name, container name, and object name are all used to determine the partition where this object should live. One or many devices is responsible for a partition. In OpenStack Swift, a storage node typically consists of multiple devices. Each device represents a physical storage unit, such as a hard disk drive (HDD) or a solid-state drive (SSD). The devices within a storage node collectively contribute to the available storage capacity and performance of that node. The devices within a storage node are commonly referred to as "drives" in Swift terminology. These drives are configured to store the data objects and provide fault tolerance and redundancy through replication.

Then a lookup in the ring figures out which storage nodes contain the partitions in question.

The data is then sent to each storage node where it is placed in the appropriate partition. At least two of the three writes must be successful before the client is notified that the upload was successful.

Next, the container database is updated asynchronously to reflect that there is a new object in it.


### Resources
https://docs.openstack.org/swift/latest/overview_ring.html
https://docs.openstack.org/swift/pike/admin/objectstorage-components.html