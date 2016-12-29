:tocdepth: 1

Overall Review
==============

There are several benefits to provide Containerization of Services for LSST that are well summarized in (ref). Separation of Roles, reduction of deployment efforts, Dynamic Service relocation, horizontal and vertical scaling, reproducibility, configuration manager versus system administrator, distribution are some of these key benefits.

`Kubernetes <http://kubernetes.io/>`_ is a flexible infrastructure for managing containerized services in one or more cluster environments. It was initially developed by Google and now it is a open source project hosted by the `Cloud Native Computing Foundation <https://www.cncf.io/>`_ (CNCF)  which is a nonprofit organization part of the Linux Foundation.

Kubernetes provide a high level, fault tolerant,  container management system across a cluster of nodes which is extensible by design accessed using a powerful API. It is one of the most used, if not the one,  container manager within the open source and industry container communities. It is one of the top open source active github projects, with more than half of contributions coming from outside Google in a constantly growing community. It provides several advantages over other container and orchestration managers such Swarm, Mesos or Nomad (Review below).

There are several features that make Kubernetes a prefered tool as a container manager, some of them are described later in this document in more detail but to list a few:

- Easy to deploy, scale and roll upgrade applications and services
- Multi tenant IaaS and PaaS (Infrastructure and Platform)
- Isolated environments.  Can run same container in prod/devel deployments.
- Provide a reliable and resilience framework, self-healing
- Efficient use of cluster resources, integrated with cloud providers
- Extensible and modular designed
- Declarative and powerful API
- Auto-replication, Auto-scaling and Load Balancing
- Completely open source, countless services on top of it (monitoring, logging, visualization, etc)
- Role Based Access Control for resources
- High Availability Multi Cluster and Cluster Federation [#f1]_
- Self Hosted Architecture [#f1]_

Infrastructure example 
======================

The schema below shows a simplified version of the current picture of a cloud infrastructure, susceptible to change, where Kubernentes is one of the main components. From the top to the bottom are the different level components in a cloud-like environment. There might be missing components and the schema it is just to put things into context. Inside the parentheses are examples of technologies associated with each component that I am more familiar with.

  .. figure:: /_static/schema.png
     :name: schema

     Cloud infrastructure 

As an example micro service, consider a JupyterHub application (which provides jupyter notebook servers to users) can run from a Kubernetes Pod  (Jupyterhub itself runs from a docker container) which creates and schedule (using Load Balancing) single pods on the nodes available for this service (using RBAC) after user is authenticated against LDAP. The nodes and the application itself are managed by Kubernetes. All data (including sessions and notebooks) are persisted in a shared File System (GPFS) with access to users home directories. When the user ends its session (or it disconnect) the pod is destroyed, the resources a freed and the data is already available for next session.

Kubernetes makes sure the JupyterHub deployment is running (by using Replication Control) as well as the pods for every individual session. In the case of many users, a larger number of nodes can be assigned to the service and new pods will be scheduled there. 

A simple schematic representation of this example is below, where 6 nodes in a cluster are used by the service, 2 pods are constantly alive managing external request by a single IP. After authentication (not shown) a single pod is created in the ‘least’ busy  node with access to the user’s GPFS home folder, the data is persisted and the pod is deleted after used. The picture below shows 8 active users and a new node about to be added to the service. Note that multiple pods can run on any node, including the jupyterhub pods or pods from other services that have access to any given node.  The status of the cluster, i.e., pods being alive, deployment status, replication status, node status is handled by a etcd key value storage which itself is replicated (not shown) in the cluster. Everything from the service being exposed, the scheduling, the balancing and the etcd storage is managed by kubernetes. 

  .. figure:: /_static/Jupyter.png
     :name: jupyter

     JupyterHub example configuration

Comparison between Container management systems
===============================================

There are several ways to control, manage and configure an isolated cluster or a cluster in a cloud computing environment. Adopting a containerized solution for running services and applications I have only considered those cluster managements like technologies that deals with deploying, scheduling and scaling docker containers within a cluster. Other popular, open sourced software for orchestration containers and ar mentioned here are: Mesos, Swarm, Fleet and Nomad.

This simple comparison is a starting point for a more in-depth comparison that can be done in the future. Only Kubernetes has been installed and tested, so the information collected here comes from software documentation and from previous experience. 


Kubernetes
^^^^^^^^^^

From their documentation:

“Kubernetes is an open-source production grade container orchestration system for automating deployment, scaling, and management of containerized applications. Designed on the same principles that allows Google to run billions of containers a week, Kubernetes can scale without increasing your ops team. Kubernetes flexibility grows with you to deliver your applications consistently and easily no matter how complex your need is. Kubernetes is open source giving you the freedom to take advantage of on-premise, hybrid, or public cloud infrastructure”

Kubernetes was originally developed by Google and made open sourced recently where may other companies took leadership and continue building, adding features and improving it to the point it is today.

It has a powerful and declarative API to access, deploy, manage and orchestrate containers in a cluster. The minimum schedule unit is called a Pod which consist in one or more container linked together creating a micro services.

These pods are ephemeral and can be deploy directly (similarly as running a container) or can be deployed by an Kubernetes Object called Deployment (or Replica Controller) which is in charge to deploy and maintain a desired state (i.e., number of pods running), autoscale (increase or decrease number of identical pods) if enabled and perform rolling upgrades without taking the service down.

Kubernetes has version control for deployment in its workload management yet (e.g., going back to a specific deployment version), which are handled by the Deployments. 

On top of each deployment, Services can be created to expose a deployment using a internal (and external) IP for accessing the pods. An internal Load Balancer distribute the required work among pods running on different nodes.

These deployments also support Multi-schedule and QoS tiers of simple usage, this means pods can be scheduled by different schedulers on a particular available node  or a node that fit certain given requirements based on QoS. This allow flexibility and efficient use of available resources.

The default network model in Kubernetes is flat and permits all pods to talk to each other. Containers in the same pod share an IP and can communicate using ports on the localhost address.

All coordination and service discovery is done using a etcd clusters which itself can be set up to be encrypted and high available, by using a RAFT consensus algorithm. This means all the information about the status and health of the cluster is being stored in multiple etcd nodes managed by a master node (usually the same master managing Kubernetes) but in case the master or the nodes are unhealthy this are replaced effortlessly to keep the system operating at all times. 


Kubernetes provide a High Availability cluster for nodes and for master as well as for pods running on the nodes. This allows the creation of fault tolerant services and infrastructure and resiliency of services.

Kubernetes also provide a secure and efficient way to share Secrets (password files, certificates) within the cluster and within the services accounts that facilitate configuration and deployment. 

It supports the creation of Volumes and Persistent Volumes that can be attached to the pods at deployment time,  by using Persistent Volume Claims. This means that even pods are being replicated, destroyed and deploy data can be persisted by using the mounted Persistent Volumes which for some applications, especially Stateful Applications (which are also supported via PetSets)  is vital for the continuous operation. PetSet in Kubernetes allow to deploy Stateful Applications and keep the same name (indexed) for every pod created (or replicated).

Kubernetes supports and provide plugins for  different volume  and object store technologies from different cloud providers which makes it very flexible. 

Similarly Kubernetes supports configuration management for the pods and deployment, which means it is possible to create a configuration file (called ConfigMap) which is stored as object in etcd and can be referenced by any pod in terms of Volumes. Which is a similar process to handle secrets, this way and deployment time pods will have access to a ConfigMap which is use to run the application, among other things.

It supports multi-tenant services, this means several services accounts can be created with a limited number of resources and usage. Authentication and authorization is done using RBAC (Role Base Access Control) which can be used on the service accounts to allow certain groups to schedule pods or limited places.

Recently Kubernetes is supporting Federation Cluster which will allow to have multiple clusters or data centers be centralized managed. Federation creates a mechanism for multi-cluster geographical replication, keeping the most critical services running even in the face of regional connectivity or data center failures.

There is an increasing development in making installation and setup of Kubernetes cluster easier, even for production environments, this was a limiting factor  when trying to deploy Kubernetes on a cloud provider. Now, it's becoming a much simpler task and multiples clusters can be created to keep production and development separated. 

There are current efforts for Kubernetes to be self-hosted which means, that after an initial bootstrapping of a cluster, the cluster itself will be self managed and can be upgraded without taking the cluster down or without backing it up on a second cluster.


Swarm
^^^^^

From Swarm documentation:

“Docker Swarm is native clustering for Docker. It turns a pool of Docker hosts into a single, virtual Docker host. Because Docker Swarm serves the standard Docker API, any tool that already communicates with a Docker daemon can use Swarm to transparently scale to multiple hosts”

Since its native for Docker, their integration is natural and great (other container technologies not supported). Swarm provides an API which sits on tip of  the Docker API which makes it transparent and easy to the user but limited to the capabilities of the Docker API. It is very simple to use and once the cluster is set up, running Swarp is almost the same as running docker containers but these containers are scheduled on different nodes.

Although is very simple to use, it doesn’t support more complex scheduling than the ones provided by Docker, nor multi-tenant services. 

It has build-in data store (go-memdb) and raft algorithm for consensus (similar to etcd) and the design correspond to a manager who is responsible of orchestration and scheduling using control loop driven orchestration, workers use pull model to connect with manager, given that a swarm container is installed on all the nodes, then just by exposing the corresponding port and ip nodes can connect to the swarm cluster

It doesn’t allocate volumes automatically (it is under development) nor link containers running on different servers. This is changing for Docker above 1.9 where persistent volumes are better handled and multi-host networking will solve swarm network limitations. 

Swarm discovery tools can be replaced by etcd or any other tools, like most of its components except for using Docker and its API. It also needs to have docker deamon running on all nodes. 

Swarm itself rely on Docker development, as Docker include new features Swarm is improved as well, for now there is not a similar Replication control from Kubernetes or monitoring tools. You can build (or use existent) graphical UI to monitor the containers and the status of the nodes although the information is limited and not multi cluster is supported (i.e., namespaces).

Swarm has not version control for deployment in its workload management yet (e.g., going back to a specific deployment version), which is implemented as a service. It can control the number of replicas but any deployment will be a new one.

 It does not have a way to manage Secrets (of all kinds, from certificates to user/password). Its under development and currently the workaround is to use a database or a similar approach to create, update and distribute secrets. 

It doesn’t have a Configuration Manager either (in kubernetes configuration is created in terms of volumes mounted to the pods), in Swarm it needs to be added ‘manually’ to the images.

It doesn’t provide auto scaling, for example when many request are being made and can be handle the system doesn’t auto scale to alleviate the problem.

Swarm uses internal Load balancing  using ipvs NAT mode and a routing mesh to expose services externally to the cluster. Different services need to be rerouted manually using a reverse proxy if needed. 

In Swarm a single container is minimal schedule unit as opposed using pods in Kubernetes and it doesn’t support multi schedulers (not labeling to schedule certain pods to a certain node) and doesn’t provide yet QoS tiers.

Swarm is usually considered the closest option for container management (and very often the top option) within the container community, its native support for Docker and its active develop, easy to set up, easy to use (especially if already familiar with Docker) makes it a very viable option. 

Mesos
^^^^^

From their documentation:

“Apache Mesos abstracts CPU, memory, storage, and other compute resources away from machines (physical or virtual), enabling fault-tolerant and elastic distributed systems to easily be built and run effectively.”

Mesos is an Apache project design to run on large scale system with multiple nodes, it is by design a resilience and high availability open source cluster manager. By itself it can’t orchestrate containers but only by using Marathon which is a container platform for Mesos (or a similar compatible infrastructure), since its 1.0 release now Mesos support a unified containerizer that support multiple container technologies in one object.

It has been shown that Mesos scales extremely well for over 10,000 nodes and recently it supports GPU as well.

The design in Mesos consist in a Master node, some slaves or Agent nodes and a Zookeeper which maintains information regarding the cluster. Multiple instances of both Master and Zookeeper are kept alive to ensure High availability and to avoid single point failures.  

It has a two level scheduling as opposed to Kubernetes in which the schedule is driven by control loops. This means that the Agents notify the master about their resources and based on allocation policy and QoS tiers the master decides which Framework will get such resources by offering them to it. Frameworks schedule the tasks and containers and run them on the nodes. Frameworks decide whether to accept such resources and if those are accepted the Framework takes over and schedule one or more task on those resources. There are multiple Frameworks (controlling containers, Hadoop, Spark, etc) and multiple resources can be allocated to a given Framework.

Mesos can run multiple containers including Docker and ACI, it doesn’t need to have a docker deamon running and as Kubernetes the deployment of these container support versioning and rolling upgrades.

Currently Mesos doesn’t not support Configuration Management for containers and has limited support for Secrets exposed through environmental variables. It can be done through volumes and persistent volumes where configuration data is stored, however it would require a predefined configuration of paths for the container.

Despite being very scalable, Mesos doesn’t provide a native way to auto scale the number of replicas for each container in an automatic way, there are workarounds this fact and can be done by monitoring the metrics directly from Mesos, through the Master and the Frameworks.

Like Kubernetes, Mesos also support deploying stateful services and applications naturally through their Frameworks and persistent volumes.

Given the Mesos design for handling large amount of data and request, it also provides a Service Discovery (for containers) using internal DNS, the same applies for Load Balacing the Services.

Overall Mesos is a robust and powerful cluster manager which have similar characteristics to Kubernetes although from a different application perspective. In can manage and orchestrate containers by using Marathon but Mesos it self was not designed for that scope. However, it can run and schedule not only jobs from inside containers but also in form of a cluster using Hadoop or Spark. 

Fleet
^^^^^

Fleet is a system that builds on top of systemd developed by CoreOS. From their documentation:

“This project is quite low-level, and is designed as a foundation for higher order orchestration. fleet is a cluster-wide elaboration on systemd units, and is not a container manager or orchestration system. fleet supports basic scheduling of systemd units across nodes in a cluster”

Fleet is a clean and a simple way to manage a cluster as if it shared a single init system. It provides a similar replication control and Load Balancer as Kubernetes to keep the container running. It is very well integrated with Docker and it's native to CoreOS. 

Although it is a powerful resource that not only manage containers but anything else in term of systemd and it is very easy to use and to configure it is too simple tool for cluster and container management and orchestration. It is a recommended tool for a small deployment projects on a fix cluster that doesn’t require all the complexity of Kubernetes, its fault-tolerant design makes this tool very robust and reliable. It has interesting features given that everything is control by systemd, among this features the ability of an API activation using sockets only which can reduce the usage of resources is very promising  but stated by their documentation fleet provides the foundation for other more complex tools and it is not designed for large scale projects.

Nomad
^^^^^

Nomad is rather new solution alternative for Kubernetes. From their documentation

“Nomad is a cluster manager, designed for both long lived services and short lived batch processing workloads. Developers use a declarative job specification to submit work, and Nomad ensures constraints are satisfied and resource utilization is optimized by efficient task packing. Nomad supports all major operating systems and virtualized, containerized, or standalone applications.”

I’ve learned about Nomad much later and since is a recent software I couldn’t dig much deeper. However, these are my notes:

It is very simple to setup and use, similar to Swarm, maybe even simpler.

Like Kubernetes, Nomad is also written in go but unlike Kubernetes it doesn’t only support Docker ( or rkt) but also virtualized, containerized and standalone applications. 

In terms of design it is much simpler than Kubernetes as only binaries are needed on every node.Nomad only aims to provide cluster management and scheduling, while Kubernetes is much bigger and higher level in scope (secrets, storage, service discovery, multi-tenant, etc)

According to their documentation, it has been tested on over 5,000 nodes and it also supports multi-datacenters and multi-regions configurations.

There is no much information regarding Persistent Volumes for Nomad, besides what can be done manually using Docker and attached volumes to the nodes.

It seems to be a more direct competitor to Swarm than Kubernetes in terms of their scope, which is much focused but limited than Kubernetes.

Summary
^^^^^^^

The table below tries to summarizes some of the features needed for a container and cluster management. Given the high level of development and community interest Kubernetes seems to be the leading technology that can fit most of the user cases. Nomad is gaining popularity but despite its potential it not as complete as other similar tools yet. Swarm and Mesos are probably the ideal solution for some specific applications. Swarm is also being heavily developed and its Docker native which can be very advantageous,  Mesos on the other hand is a well known big data solution, especially for Map Reduce and similar problems, that can also run container applications. Their approach is somehow different and more complex than Kubernetes but it seems to be a closer competitor to Kubernetes, both show advantages and disadvantages which will depend on the specific use of the cluster. Usually is the problem and the data is known, Mesos can be a very good candidate, on the other hand, if different datasets are used by different algorithms that scale differently Kubernetes provide an excellent solution, especially for large development groups with different roles within datacenters, like LSST.



+--------------+-------------+---------------+-------------+-------------+-----------+
|              | Kubernetes  | Swarm         | Mesos       | Nomad       | Fleet     |
+==============+=============+===============+=============+=============+===========+
|  Set up      | Medium      | Easy          | Medium      | Easy        | Easy      |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Containers   | Docker + rkt| Docker        || Yes, using || Standalone | Docker    |
|              |             |               || Marathon   || +containers|           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Coordination | etcd        || Internal RAFT| ZooKeeper   || External   | etcd      |
|              |             || optional     |             || Consul     |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Strategy     || Control    || Control      || 2 level    || Control    | serviced  |
|              || loop driven|| loop driven  || scheduling || loop driven|           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| API          || Declarative| Same as Docker|| Declarative| Simple      | Simple    |
|              || powerful   |               || powerful   |             |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
|| High        | Yes         | Almost        | Yes         | Not sure    | No        |
|| Availability|             |               |             |             |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Resilience   | Yes         | Possible      | Yes         | Not sure    | No        |
+--------------+-------------+---------------+-------------+-------------+-----------+
|| Multi       | Yes         | No            | Yes         | Not sure    | No        |
|| Scheduler   |             |               |             |             |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Schedule unit| Pod         | Container     | Task/Docker | Task        | Container |
+--------------+-------------+---------------+-------------+-------------+-----------+
|| Service     | Yes         | No            | Yes         | Yes         | Workaround|
|| Discovery   |             |               |             |             |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
||  External   | Yes, several| Routing mesh  | HAproxy     | Manual      | Workaround|
||  Access     |             |               |             |             |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| GUI Monitor  | Included    | External      | Included    | External    | External  |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Replication  | Yes         | Yes           | Yes         | Yes         | Yes       |
+--------------+-------------+---------------+-------------+-------------+-----------+
|| Rolling     | Yes         | No            | Yes         | No          | No        |
|| Upgrades    |             |               |             |             |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Secrets      | Yes         | Not yet       || No, only   || External   | No        |
|              |             |               || Workaround || Vault      |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
|| Persistent  | Yes         | Is possible   | Yes         | Workaround  | Possible  |
|| Volumes     |             |               |             |             |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| QoS tiers    | Yes         | Not yet       | Yes         | Not yet     | No        |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Scalability  | 3,000 nodes |               | 10,000 nodes| 5,000 nodes |           |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Multi tenant | Yes         | No            | Yes         | No          | No        |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Auto-Scaling | Yes         | No            | Is possible | No          | No        |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Membership   | etcd        | gossip        | Yes         | gossip      | No        |
+--------------+-------------+---------------+-------------+-------------+-----------+
| Auth         | RBAC        | Machine users | credentials | Yes         | Possible  |
+--------------+-------------+---------------+-------------+-------------+-----------+

.. note::

   **This technote is not yet published.**

   A short description of this document

.. rubric:: Footnotes

.. [#f1] Currently under active development
