## Glance 

The Image service (glance) enables users to discover, register, and retrieve virtual machine images. It offers a REST API that enables you to query virtual machine image metadata and retrieve an actual image. You can store virtual machine images made available through the Image service in a variety of locations, from simple file systems to object-storage systems like OpenStack Object Storage.

The OpenStack Image service is central to Infrastructure-as-a-Service (IaaS). It accepts API requests for disk or server images, and metadata definitions from end users or OpenStack Compute components. It also supports the storage of disk or server images on various repository types, including OpenStack Object Storage.

A number of periodic processes run on the OpenStack Image service to support caching. Replication services ensure consistency and availability through the cluster. Other periodic processes include auditors, updaters, and reapers.

Glance is not running inside Apache, but is an independent process using a standalone WSGI server. To understand the startup process, let us start with the setup.cfg file. This file contains an entry point glance-api which, via the usual mechanism provided by Pythons setuptools, will provide a Python executable which runs glance/cmd/api.py. This in turn uses the simple WSGI server implemented in glance/common/wsgi.py. Here we see that the actual WSGI app is created and passed to the server using the PasteDeploy Python library. If you have read my previous post on WSGI and WSGI middleware, you will know that this is a library which uses configuration data to plumb together a WSGI application and middleware. The actual call of the PasteDeploy library is delegated to a helper library in glance/common and happens in the function load_past_app defined here.


The OpenStack Image service includes the following components:
- glance-api: accepts Image APPI calls for image discovery, retrieval and storage
- glance-registry: Stores, processes, and retrieves metadata about images. Metadata includes items such as size and type.
- Database: Stores image metadata and you can choose your database depending on your preference. Most deployments use MySQL or SQLite.
- Storage repository for image files: Various repository types are supported including normal file systems (or any filesystem mounted on the glance-api controller node), Object Storage, RADOS block devices, VMware datastore, and HTTP. Note that some repositories will only support read-only usage.
- Metadata definition service: A common API for vendors, admins, services, and users to meaningfully define their own custom metadata. This metadata can be used on different types of resources like images, artifacts, volumes, flavors, and aggregates. A definition includes the new property’s key, description, constraints, and the resource types which it can be associated with.

Image statuses
https://docs.openstack.org/glance/victoria/user/statuses.html 

Disk and Container Formats
https://docs.openstack.org/glance/victoria/user/formats.html 

Image signature verification
https://docs.openstack.org/glance/victoria/user/signature.html 

### WSGI 
Glance uses the PasteDeploy mechanism to create a WSGI application. When you take a look at the corresponding configuration, however, you will see that it contains a variety of different pipeline definitions. To select the pipeline that will actually be deployed, Glance has a configuration option called deployment flavor. This is a short form for the name of the pipeline to be selected, and when the actual pipeline is assembled here, the name of the pipeline is put together by combining the flavor with the string “glance-api”. We use the flavor “keystone” which will result in the pipeline “glance-api-keystone” being loaded.

This pipeline contains the Keystone auth token middleware which (as discussed in our deep dive into tokens and policies) extracts and validates the token data in a request. This middleware components needs access to the Keystone API, and therefore we need to add the required credentials to our configuration in the section [keystone_authtoken].

### Keywords 
- Image: a file that contains the OS, your executable, and any data files that might be related to your programs. (It is raw data without the resources like CPU and memory).

### Resources: 
https://leftasexercise.com/2020/02/10/openstack-supporting-services-glance-and-placement/ 


