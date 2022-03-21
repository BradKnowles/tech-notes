---
tags: [docker/orchestration,container/orchestration,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Understanding orchestration: Docker Swarm and Kubernetes

The same images in lower environments will be used in production, but the management layer coordinating the container activities will be different.  This management layer is called [[orchestration]].  [[Docker Swarm]] and [[Kubernetes]] are the two main players in the space, with Kubernetes having the lion's share.   However, it can be complex to learn and use.  Docker Swarm is easier to learn and has lots of the same concepts.

## 12.1 - What is a container orchestrator?

An orchestrator is multiple machines grouped together form a [[cluster]].  The orchestrator manages containers, distributing work among all the machines, [[load balancing]] network traffic, and replacing containers that become unhealthy.

You interact with the cluster as a single unit, sending commands and running queries through the API.  The cluster could be 1,000 machines or a single machine.

## 12.2 - Setting up a Docker Swarm cluster

```powershell
docker swarm init
```
<pre>
Swarm initialized: current node (7tnjqs26p1dpuvrqclcf6gst3) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0hwp449iuvbpo27ly5uxnej5dik44zv8ipw2zfvzc0j6a23a54-bo1qnupu8oye1bo7wxtfymr90 &ltip address redacted&gt:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
</pre>

This created a cluster with a single node as the manager.  Managers are in charge of the cluster.  The cluster database is stored on the managers, the commands you issue are sent to the API running on the managers, and the scheduling and monitoring is all done by the managers.

Workers just run containers according to the manager's schedule and report back the status.

> Managers can also run workloads.

Initializing Swarm is done once and any number of machines, or nodes, can join.  To join a swarm:
- The node has to be on the same network
- The join token from the manager is required.

If you have manager access, you can use that to print the tokens and see a list of all nodes in the swarm.

```powershell
# print the command to join a new worker node
docker swarm join-token worker

# print the command to join a new manager node
docker swarm join-token manager

# list all the nodes in the swarm
docker node ls
```

A single-node Swarm works exactly the same way as a multi-node Swarm, except you don't get high availability from having multiple machines or the option to scale out containers to use the capacity of many machines.

> For a highly available Swarm, you need multiple managers, typically three.

![[MultiNodeSwarm.jpg]]

## 12.3 - Running applications are Docker Swarm services

With Docker Swarm you deploy services, not containers.  The Swarm runs the containers for you.  A service can be deployed as multiple containers.

Services are defined using a lot of the same information as containers; image to use, environment variables, ports, and a name for the services that becomes its [[Domain Name System|DNS]] on the network.  Services can have many replicas, which are individual containers that all use the same specification from the service and can be run on any node in the Swarm.

```powershell
docker service create --name timecheck --replicas 1 diamol/ch12-timecheck:1.0

docker service ls
```
<pre>
ubwyjob8t7cp1l7rhz6ayizoz
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged

ID             NAME        MODE         REPLICAS   IMAGE                       PORTS
ubwyjob8t7cp   timecheck   replicated   1/1        diamol/ch12-timecheck:1.0
</pre>

> Services are first-class objects in Docker Swarm, but you need to be running in Swarm mode--or be connected to a Swarm manager--to work with them.

Replicas are just the ordinary Docker containers that make up a service.  Normal container commands will work on replicas.

> It’s not something you would normally do, though, because the containers are being managed by the Swarm. If you try to manage them yourself, what happens may not be what you expect.

```powershell
# list the replicas for the service:
docker service ps timecheck

# check the containers on the machine:
docker container ls

# remove the most recent container (which is the service replica)
docker container rm -f $(docker container ls --last 1 -q)

# check the replicas again:
docker service ps timecheck

# check the containers again:
docker container ls
```
<pre>
PS> docker service ps timecheck
ID             NAME          IMAGE                       NODE             DESIRED STATE   CURRENT STATE           ERROR     PORTS
6y1jwd859idb   timecheck.1   diamol/ch12-timecheck:1.0   docker-desktop   Running         Running 4 minutes ago

PS> docker container ls
CONTAINER ID   IMAGE                       COMMAND                  CREATED         STATUS         PORTS     NAMES
<span style="color: blue">e67f3807b8d6</span>   diamol/ch12-timecheck:1.0   "dotnet TimeCheck.dll"   4 minutes ago   Up 4 minutes             timecheck.1.6y1jwd859idb0bt66w2pd0r8f

PS> docker container rm -f $(docker container ls --last 1 -q)
e67f3807b8d6

PS> docker service ps timecheck
ID             NAME              IMAGE                       NODE             DESIRED STATE   CURRENT STATE          ERROR                         PORTS
lxclzk9ol2t0   timecheck.1       diamol/ch12-timecheck:1.0   docker-desktop   Running         Running 1 second ago
6y1jwd859idb    \_ timecheck.1   diamol/ch12-timecheck:1.0   docker-desktop   Shutdown        Failed 6 seconds ago   "task: non-zero exit (137)"

PS> docker container ls
CONTAINER ID   IMAGE                       COMMAND                  CREATED         STATUS         PORTS     NAMES
<span style="color: #f88017">fb54272f5655</span>   diamol/ch12-timecheck:1.0   "dotnet TimeCheck.dll"   2 minutes ago   Up 2 minutes             timecheck.1.lxclzk9ol2t0yyawl5ozhdz5c
PS>
</pre>

The service was running 1 replica, container ID `e67f3807b8d6`.  We manually stopped the container, and Docker Swarm spun up another container, `fb54272f5655` automatically to replace it.

With Swarm mode applications are managed as services and Swarm deals with the individual containers.
```powershell
# print the service logs for the last 10 seconds:
docker service logs --since 10s timecheck

# get the service details, showing just the image:
docker service inspect timecheck -f '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
```
<pre>
PS> docker service logs --since 10s timecheck
timecheck.1.lxclzk9ol2t0@docker-desktop    | App version: 1.0; time check: 21:19.00
timecheck.1.lxclzk9ol2t0@docker-desktop    | App version: 1.0; time check: 21:19.05

PS> docker service inspect timecheck -f '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
diamol/ch12-timecheck:1.0@sha256:9d3010a572344c988da8e28444ed345c63662a5c211886e670a8ef3c84689b4e
</pre>

A major difference between Docker Swarm and Docker Compose is Swarm stores the app definition in the cluster.  With Compose you always need access to the local YAML files, with Swarm you do not.

```powershell
# update the service to use a new application image
docker service update --image diamol/ch12-timecheck:2.0 timecheck

# list the service replicas:
docker service ps timecheck

# and check the logs
docker service logs --since 20s timecheck
```
<pre>
PS> docker service update --image diamol/ch12-timecheck:2.0 timecheck
timecheck
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged

PS> docker service ps timecheck
ID             NAME              IMAGE                       NODE             DESIRED STATE   CURRENT STATE            ERROR                         PORTS
n90qm8ffb7z4   timecheck.1       <span style="color: #f88017">diamol/ch12-timecheck:2.0</span>   docker-desktop   Running         Running 8 seconds ago
lxclzk9ol2t0    \_ timecheck.1   diamol/ch12-timecheck:1.0   docker-desktop   Shutdown        Shutdown 9 seconds ago
6y1jwd859idb    \_ timecheck.1   diamol/ch12-timecheck:1.0   docker-desktop   Shutdown        Failed 18 minutes ago    "task: non-zero exit (137)"

PS> docker service logs --since 20s timecheck
timecheck.1.n90qm8ffb7z4@docker-desktop    | <span style="color: #f88017">App version: 2.0</span>; time check: 21:25.42
timecheck.1.n90qm8ffb7z4@docker-desktop    | <span style="color: #f88017">App version: 2.0</span>; time check: 21:25.47
timecheck.1.n90qm8ffb7z4@docker-desktop    | <span style="color: #f88017">App version: 2.0</span>; time check: 21:25.52
timecheck.1.n90qm8ffb7z4@docker-desktop    | <span style="color: #f88017">App version: 2.0</span>; time check: 21:25.57
</pre>

Notice the new container is tagged with `2.0` and the logs are showing `App version: 2.0` as well.

> Automated rolling updates are a huge improvement on manual application releases, and they’re another feature for supporting self-healing applications. The update process checks that new containers are healthy as it is rolling them out; if there’s a problem with the new version, and the containers are failing, the update can be automatically paused to prevent breaking the whole application. Swarm also stores the previous specification of a service in its database, so if you need to manually roll back to the previous version, you can do that with a single command.

```powershell
# rollback the previous update:
docker service update --rollback timecheck

# list all the service replicas:
docker service ps timecheck

# print the logs from all replicas for the last 25 seconds:
docker service logs --since 25s timecheck
```
<pre>
PS> docker service update --rollback timecheck
timecheck
rollback: manually requested rollback
overall progress: rolling back update: 1 out of 1 tasks
1/1: running   [>                                                  ]
verify: Service converged

PS> docker service ps timecheck
ID             NAME              IMAGE                       NODE             DESIRED STATE   CURRENT STATE            ERROR                         PORTS
krqjv2fybey1   timecheck.1       <span style="color: #f88017">diamol/ch12-timecheck:1.0</span>   docker-desktop   Running         Running 6 seconds ago
n90qm8ffb7z4    \_ timecheck.1   diamol/ch12-timecheck:2.0   docker-desktop   Shutdown        Shutdown 6 seconds ago
lxclzk9ol2t0    \_ timecheck.1   diamol/ch12-timecheck:1.0   docker-desktop   Shutdown        Shutdown 7 minutes ago
6y1jwd859idb    \_ timecheck.1   diamol/ch12-timecheck:1.0   docker-desktop   Shutdown        Failed 25 minutes ago    "task: non-zero exit (137)"

PS> docker service logs --since 25s timecheck
timecheck.1.krqjv2fybey1@docker-desktop    | <span style="color: #f88017">App version: 1.0</span>; time check: 21:32.41
timecheck.1.krqjv2fybey1@docker-desktop    | <span style="color: #f88017">App version: 1.0</span>; time check: 21:32.46
timecheck.1.n90qm8ffb7z4@docker-desktop    | App version: 2.0; time check: 21:32.27
timecheck.1.n90qm8ffb7z4@docker-desktop    | App version: 2.0; time check: 21:32.32
</pre>

Notice the `1.0` container is running and the `2.0` container has stopped and the `App version`s are back to `1.0`.

## 12.4 - Managing network traffic in the cluster
Docker Swarm does all kind of magic to make cross-cluster networking work.  Fortunately, it's not necessary to know all that because things just work.

Swarm creates an new `overlay` network.  It's a virtual network that spans *all the nodes* in the cluster.  When services are attached to the overlay, they can communicate with each other using the service name as the [[Domain Name System|DNS]] name.

### VIP Networking
Another difference between Swarm and Compose is how they handle IP Addresses.  Compose will return the IP addresses for all the containers for a DNS query while Swarm returns a **single virtual IP address** for a service.

Setting up a new service using the Image of the Day from earlier chapters
```powershell
# remove the original app:
docker service rm timecheck

# create an overlaynetwork for the new app:
docker network create --driver overlay iotd-net

# create the API service, attaching it to the network
docker service create --detach --replicas 3 --network iotd-net --name iotd diamol/ch09-image-of-the-day

# and the log API, attached to the same network
docker service create --detach --replicas 2 --network iotd-net --name accesslog diamol/ch09-access-log

# check the services
docker service ls
```

<pre>
PS C:\Users\Brad\Code> docker service rm timecheck
timecheck

PS C:\Users\Brad\Code> docker network create --driver overlay iotd-net
inn9b0uz6ukhesca0b2q8n92q

PS C:\Users\Brad\Code> docker service create --detach --replicas 3 --network iotd-net --name iotd diamol/ch09-image-of-the-day
vpcuv6j0y1qpc9uxvwbfy3tko

PS C:\Users\Brad\Code> docker service create --detach --replicas 2 --network iotd-net --name accesslog diamol/ch09-access-log
y3wb7bjiacpug2472vau3fepe

PS C:\Users\Brad\Code> docker service ls
ID             NAME        MODE         REPLICAS   IMAGE                                 PORTS
y3wb7bjiacpu   accesslog   replicated   2/2        diamol/ch09-access-log:latest
vpcuv6j0y1qp   iotd        replicated   3/3        diamol/ch09-image-of-the-day:latest
</pre>

With the app running, we'll connect to a container to run some DNS queries to demonstrate.

```powershell
# run a terminal session - Windows containers:
docker container exec -it $(docker container ls --last 1 -q) cmd

# run a terminal session - Linux containers:
docker container exec -it $(docker container ls --last 1 -q) sh

# run DNS lookups
nslookup iotd
nslookup accesslog
```
<pre>
PS C:\Users\Brad\Code> docker container exec -it $(docker container ls --last 1 -q) sh
/app # nslookup iotd
nslookup: can't resolve '(null)': Name does not resolve
<span style="color: blue">
Name:      iotd
Address 1: 10.0.1.2
</span>
/app # nslookup accesslog
nslookup: can't resolve '(null)': Name does not resolve
<span style="color: blue">
Name:      accesslog
Address 1: 10.0.1.7
</span>
/app #
</pre>

This single IP address stays constant even when the service is scaled up or down.  The networking layer decides which container instance to send traffic to, while clients keep using the same IP address.

### Ingress Networking

Swarm mode uses the same approach to handle traffic coming into the cluster.  Every node listens on the same port externally and Docker directing traffic internally within the cluster.

Creating the web front end using ingress networking
```powershell
# create the web front end of the app:
docker service create --detach --name image-gallery --network iotd-net --publish 8010:80 --replicas 2 diamol/ch09-image-gallery

# list all services:
docker service ls
```

<pre>
PS> docker service create --detach --name image-gallery --network iotd-net --publish 8010:80 --replicas 2 diamol/ch09-image-gallery
k7mss23x9uy3e3ij4x8x8bpa3

PS> docker service ls
ID             NAME            MODE         REPLICAS   IMAGE                                 PORTS
y3wb7bjiacpu   accesslog       replicated   2/2        diamol/ch09-access-log:latest
k7mss23x9uy3   image-gallery   replicated   2/2        diamol/ch09-image-gallery:latest      <span style="color: blue">*:8010->80/tcp</span>
vpcuv6j0y1qp   iotd            replicated   3/3        diamol/ch09-image-of-the-day:latest
</pre>
The service has two replicas but is listening on a single port `8010`.  Docker then routes it to the correct replica.

## 12.5 - Understanding the choice between Docker Swarm and Kubernetes

Docker Swarm is a simple orchestrator built upon the concepts of networks and services from Docker Compose and built into the Docker Engine.  Other orchestrators have come and gone but now the choice comes down to [[Docker Swarm]]Docker Swarm and [[Kubernetes]].

Kubernetes is the more popular option since all the cloud providers have managed offerings.  To setup Docker Swarm with the cloud providers you'd have to manage some of the infrastructure yourself.

Some points to consider:
- Infrastructure - Kubernetes is simpler in the cloud, but in a datacenter, Swarm is easier to manage.
- Learning curve - Swarm is an extension of Docker and Compose so the curve is smaller.  Kubernetes is much larger and complex due to it's enhanced feature set.
- Feature set - The reason Kubernetes is complex is because it's highly configurable.  [[Blue green deployments|Blue/green deployments]], automatic service scaling, and role-based access control are not available in Swarm.
- Future investment - Kubernetes benefits from a large open source community with changes and new features.  Swarm is considered a stable project and doesn't add new features frequently.