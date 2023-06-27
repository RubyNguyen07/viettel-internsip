## Keystone 

The OpenStack Identity service provides a single point of integration for managing **authentication**, **authorization**, and a **catalog of services**.

The Identity service is typically the first service a user interacts with. Once authenticated, an end user can use their identity to access other OpenStack services.

Users and services can locate other services by using the service catalog, which is managed by the Identity service. As the name implies, a service catalog is a collection of available services in an OpenStack deployment. Each service can have one or many endpoints and each endpoint can be one of three types: admin, internal, or public. In a production environment, different endpoint types might reside on separate networks exposed to different types of users for security reasons.

Components in Identity service: 
- Server: A centralized server provides authentication and authorization services using a RESTful interface.

- Drivers: Drivers or a service back end are integrated to the centralized server. They are used for accessing identity information in repositories external to OpenStack, and may already exist in the infrastructure where OpenStack is deployed (for example, SQL databases or LDAP servers).

- Modules: Middleware modules run in the address space of the OpenStack component that is using the Identity service. These modules intercept service requests, extract user credentials, and send them to the centralized server for authorization. The integration between the middleware modules and OpenStack components uses the Python Web Server Gateway Interface.

### Architecture 
- Services: Keystone is organized as a group of internal services exposed on one or many endpoints. Many of these services are used in a combined fashion by the frontend. 
+ Identity: provides auth credential validation and data about users and groups. In the basic case, this data is managed by the Identity service, allowing it to also handle all CRUD operations associated with this data. In more complex cases, the data is instead managed by an authoritative backend service.
+ Resource: provides data about projects and domains.
+ Assignment: provides data about roles and role assignments. 
+ Token: validates and manages tokens used for authenticating requests once a user’s credentials have already been verified.
+ Catalog: The Catalog service provides an endpoint registry used for endpoint discovery.


### Identity
The Identity Service in Keystone provides the Actors. Identities in the Cloud may come from various locations, including but not limited to SQL, LDAP, and Federated Identity Providers:
- SQL: Keystone will store information such as name, password, and description in SQL database. The settings for the database must be specified in Keystone’s configuration file. Essentially, Keystone is acting as an Identity Provider.
- LDAP: Keystone also has the option to retrieve and store your actors (Users and Groups) in Lightweight Directory Access Protocol (LDAP). Keystone will access the LDAP just like any other application that uses the LDAP (System Login, Email, Web Application, etc). The settings for connecting to the LDAP are specified in Keystone’s configuration file. 
- Multiple backends: Able to simultaneously support multiple LDAPs for various user accounts and SQL backends for service accounts. Leverage existing identity LDAP while not impacting it.
- Identity providers: Keystone is able to consume federated authentication via Apache modules for multiple trusted Identity Providers. These users are not stored in Keystone, and are treated as ephemeral. The federated users will have their attributes mapped into group-based role assignments. From a Keystone perspective, an identity provider is a source for identities; it may refer to software that is backed by various backends (LDAP, AD, MongoDB) or Social Logins (Google, Facebook, Twitter). Essentially, it is software (IBM’s Tivoli Federated Identity Manager, for instance) that abstracts out the backend and translates user attributes to a standard federated identity protocol format (SAML, OpenID Connect). 

### Authentication
Authentication with Keystone via: 
- Password 
- Token 

### Access Management and Authorization 
Managing access and authorizing what APIs a user can use is one of the key reasons Keystone is essential to OpenStack. Keystone’s approach to this problem is to create a Role-Based Access Control (RBAC) policy that is enforced on each public API endpoint. These policies are stored in a file on disk, commonly named policy.json.

### RBAC 

RBAC aims to solve the inability to share certain Neutron resources with a subset of projects or tenants. Neutron has supported shared resources in the past, but, until now it’s been all-or-nothing. If a network or other resource is marked as shared, it is shared with all tenants with no ability to specify otherwise. RBAC policies allow administrators and users to share resources with one or more tenants using a granular, rather than a shotgun, approach.


### Tokens 

Tokens are used to authenticate and authorize your interactions with OpenStack APIs. Tokens come in many scopes, representing various authorization and sources of identity. Token includes ID and payload. 


#### UUID 

UUID tokens are 32 bytes in length and must be persisted in a back end. Clients must pass their UUID token to the Identity service in order to validate it.

As mentioned above, UUID tokens must be persisted. By default, keystone persists UUID tokens using a SQL backend. An unfortunate side-effect is that the size of the database will grow over time regardless of the token’s expiration time. Expired UUID tokens can be pruned from the backend using keystone’s command line utility. 

##### Token generation workflow 

[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/ksimg/uuid-token.png)

##### Token validation workflow 

[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/ksimg/uuid-valid.png)

##### Token revocation workflow 

[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/ksimg/uuid-revocation.png)

##### Multiple data centers

UUID tokens for center A will not work for center B because tokens in backends are not synced. 

[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/ksimg/uuid-revocation.png)


#### PKI 

Tokens store: timestamps, expiry date, user profiler, project, domain, service catalog, ... in the payload. The payload is signed based on X509. For PKIz then the payload is compressed using zlib. 

Token size (including one endpoint in the service catalog) is approximately 1700 bytes (very large for bigger service catalogs).

To configure PKI token, we need to use 3 certificates: 
- Signing key to create private key 
- Signing certificates using signing key to create CSR (Certificate Signing Request) -> Submit CSR to CA to receive certificate from CA. 
- Certificate Authority Certificate

##### Token generation workflow 

[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/ksimg/kipgen.png)

##### Token validation workflow 

[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/ksimg/kpi-valid.png)

##### Token revocation workflow 

Similar to that of UUID token 

##### Multiple data centers 

For this type of token, backend databases of data centers have to be synced to support authentication and authorization. 
[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/ksimg/pki-gen.png)

#### Fernet 

A fernet token is a bearer token that represents user authentication. Fernet tokens contain a limited amount of identity and authorization data in a MessagePacked payload. The payload is then wrapped as a Fernet message for transport, where Fernet provides the required web safe characteristics for use in URLs and headers. The data inside a fernet token is protected using symmetric encryption keys, or fernet keys.

A fernet key is used to encrypt and decrypt fernet tokens. Each key is actually composed of two smaller keys: a 128-bit AES encryption key and a 128-bit SHA256 HMAC signing key. The keys are held in a key repository that keystone passes to a library that handles the encryption and decryption of tokens.

A ferney token is quite small (about 255 bytes), storing enough data to avoid being stored in the database. 

Fields in a Fernet token: 
- Fernet format version 
- Current timestamp 
- Initialization vector 
- Ciphertext 
- HMAC 

##### Fernet keys 

File fernet key /etc/keystone/fernet-keys => 0 1 2 3 4

A key repository is required by keystone in order to create fernet tokens. These keys are used to encrypt and decrypt the information that makes up the payload of the token. Each key in the repository can have one of three states. The state of the key determines how keystone uses a key with fernet tokens. The different types are as follows:

- Primary key (highest index):
There is only ever one primary key in a key repository. The primary key is allowed to encrypt and decrypt tokens. This key is always named as the highest index in the repository.

- Secondary key (index between primary key and staged key):
A secondary key was at one point a primary key, but has been demoted in place of another primary key. It is only allowed to decrypt tokens. Since it was the primary at some point in time, its existence in the key repository is justified. Keystone needs to be able to decrypt tokens that were created with old primary keys.

- Staged key (smallest index):
The staged key is a special key that shares some similarities with secondary keys. There can only ever be one staged key in a repository and it must exist. Just like secondary keys, staged keys have the ability to decrypt tokens. Unlike secondary keys, staged keys have never been a primary key. In fact, they are opposites since the staged key will always be the next primary key. This helps clarify the name because they are the next key staged to be the primary key. This key is always named as 0 in the key repository.

The fernet keys have a natural lifecycle. Each key starts as a staged key, is promoted to be the primary key, and then demoted to be a secondary key. New tokens can only be encrypted with a primary key. Secondary and staged keys are never used to encrypt token. The staged key is a special key given the order of events and the attributes of each type of key. The staged key is the only key in the repository that has not had a chance to encrypt any tokens yet, but it is still allowed to decrypt tokens. As an operator, this gives you the chance to perform a key rotation on one keystone node, and distribute the new key set over a span of time. This does not require the distribution to take place in an ultra short period of time. Tokens encrypted with a primary key can be decrypted, and validated, on other nodes where that key is still staged.


##### Token generation workflow 

[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/ksimg/fernet-gen.png)

##### Token validation workflow 

[Diagram](https://github.com/thanh474/internship-2020/raw/master/ThanhBC/Openstack/Keystone/ksimg/fernet-valid.png)

##### Multiple data centers 

As long as each keystone node shares the same key repository, fernet tokens can be created and validated instantly across nodes.


### Key terms 
- Domain: abstractions used to group and isolate resources. A collection of users, groups, and projects. Isolate the visibility of a set of Projects and Users (and User Groups) to a specific organization.  A domain can serve as a logical division between different portions of an enterprise, or each domain can represent completely separate enterprises. For example, a user named John in domain A is different from another use named John in domain B (they have different UUID - universally unique identifier)
- Project (aka Tenants): abstractions used to group and isolate resources (e.g. servers, images). Projects themselves don’t own Users, but Users or User Groups are given access to a Project using the concept of Role Assignments.
- Grant: assigning a role to a user 
- Actors: users and groups 
- Targets: Projects and Domains 
- Token: In order for a user to call any OpenStack API they need to (a) prove who they are, and (b) that they should be allowed to call the API in question. The way they achieve that is by passing an OpenStack token into the API call—and Keystone is the OpenStack service responsible for generating these tokens. A user receives this token upon successful authentication against Keystone. The token also carries with it authorization. It contains the authorization a user has on the cloud. A token has both an ID and a payload. The ID of a token is guaranteed to be unique per cloud, and the payload contains data about the user.
- Catalog: contains the URLs and endpoints of the different Cloud services. Without the catalog, users and applications would not know where to route requests to create VMs or store objects. The service catalog is broken up into a list of endpoints, and each endpoint is broken down into an admin URL, internal URL, and public URL, which may be the same.
- An unscoped token is one where the user is authenticated but not for a specific project or domain. This type of token is useful for making queries such as determining what projects a user has access to. 
- Fernet token: message packed tokens that contain authentication and authorization data. Fernet tokens are signed and encrypted before being handed out to users. Most importantly, however, Fernet tokens are ephemeral. This means they do not need to be persisted across clustered systems in order to successfully be validated.
- Scope: Scope is an overloaded term. In reference to authenticating, as seen above, scope refers to the portion of the POST data that dictates what Resource (project, domain, or system) the user wants to access. In reference to tokens, scope refers to the effectiveness of a token, i.e.: a project-scoped token is only useful on the project it was initially granted for. A domain-scoped token may be used to perform domain-related function. A system-scoped token is only useful for interacting with APIs that affect the entire deployment. In reference to users, groups, and projects, scope often refers to the domain that the entity is owned by. i.e.: a user in domain X is scoped to domain X.


- Mapping of policy target to API: https://docs.openstack.org/keystone/victoria/getting-started/policy_mapping.html 
- API examples: https://docs.openstack.org/keystone/latest/api_curl_examples.html
- Keystone configuration: https://docs.openstack.org/keystone/latest/admin/configuration.html#troubleshoot-the-identity-service 

### References 
[https://www.oreilly.com/library/view/identity-authentication-and/9781491941249/ch01.html](https://www.oreilly.com/library/view/identity-authentication-and/9781491941249/ch01.html)
[https://www.redhat.com/en/blog/introduction-fernet-tokens-red-hat-openstack-platform](https://www.redhat.com/en/blog/introduction-fernet-tokens-red-hat-openstack-platform)
[https://docs.openstack.org/keystone/victoria/getting-started/architecture.html](https://docs.openstack.org/keystone/victoria/getting-started/architecture.html)
[https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/Token-trong-keystone.md](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/Keystone/Token-trong-keystone.md)
[https://docs.openstack.org/keystone/pike/admin/identity-fernet-token-faq.html](https://docs.openstack.org/keystone/pike/admin/identity-fernet-token-faq.html)