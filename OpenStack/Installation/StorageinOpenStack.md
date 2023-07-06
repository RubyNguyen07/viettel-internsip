# Introduction to Persistent Storage in OpenStack

OpenStack recognizes two types of storage: ephemeral and persistent. Ephemeral storage is storage that is associated only to a specific Compute instance. Once that instance is terminated, so is its ephemeral storage. This type of storage is useful for basic runtime requirements, such as storing the instance’s operating system.

Persistent storage, on the other hand, is designed to survive ("persist") independent of any running instance. This storage is used for any data that needs to be reused, either by different instances or beyond the life of a specific instance. OpenStack uses the following types of persistent storage:

- Volumes: The OpenStack Block Storage service (openstack-cinder) allows users to access block storage devices through volumes. Users can attach volumes to instances in order to augment their ephemeral storage with general-purpose persistent storage. Volumes can be detached and re-attached to instances at will, and can only be accessed through the instance they are attached to.
Volumes also provide inherent redundancy and disaster recovery through backups and snapshots. In addition, you can also encrypt volumes for added security. For more information about volumes, see Chapter 2, Block Storage and Volumes.
Note: Instances can also be configured to use absolutely no ephemeral storage. In such cases, the Block Storage service can write images to a volume; in turn, the volume can be used as a bootable root volume for an instance.

- Containers: The OpenStack Object Storage service (openstack-swift) provides a fully-distributed storage solution used to store any kind of static data or binary object, such as media files, large datasets, and disk images. The Object Storage service organizes these objects through containers.
While a volume’s contents can only be accessed through instances, the objects inside a container can be accessed through the Object Storage REST API. As such, the Object Storage service can be used as a repository by nearly every service within the cloud. For example, the Data Processing service (openstack-sahara) can manage all of its binaries, data input, data output, and templates directly through the Object Storage service.

- Shares: The OpenStack Shared File System service (openstack-manila) provides the means to easily provision remote, shareable file systems, or shares. Shares allow tenants within the cloud to openly share storage, and can be consumed by multiple instances simultaneously.
Important
The OpenStack Shared File System service is available in this release as a Technology Preview, and therefore is not fully supported by Red Hat. It should only be used for testing, and should not be deployed in a production environment. For more information about Technology Preview features, see Scope of Coverage Details.

Each storage type is designed to address specific storage requirements. Containers are designed for wide access, and as such feature the highest throughput, access, and fault tolerance among all storage types. Container usage is geared more towards services.

On the other hand, volumes are used primarily for instance consumption. They do not enjoy the same level of access and performance as containers, but they do have a larger feature set and have more native security features than containers. Shares are similar to volumes in this regard, except that they can be consumed by multiple instances.

Differences in a table: [Table](https://docs.openstack.org/arch-design/design-storage/design-storage-concepts.html#table-openstack-storage)

# Resources
https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/8/html/storage_guide/ch-intro-persistent#:~:text=OpenStack%20recognizes%20two%20types%20of,to%20a%20specific%20Compute%20instance.
https://docs.openstack.org/arch-design/design-storage/design-storage-concepts.html