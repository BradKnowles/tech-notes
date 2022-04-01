---
tags: [docker/orchestration,container/orchestration,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Automating releases with upgrades and rollbacks

Understanding how the orchestrator of choice, [[Docker Swarm]] in this case, handles deployments and rollbacks is crucial to managing zero-downtime deployments.  Each app is different so having appropriately health checks in place so that Swarm can know how to upgrade or rollback a change becomes vital.

## 14.1 - The application upgrade process with Docker

While you may be tempted to only deploy your application when there is a new version, there are actually at least four deployment cadences that must be considered.
1.  Your Application and its dependencies
1.  The SDK used to build the application
1.  The application platform it runs on
1.  The operating system itself.

Using a .NET core app build for Linux as an example
1.  The SDK image is  from`debian:buster-slim` and contains `dotnet/core/sdk:3.0`This will be updated:
	1. The SDK version is bumped, `3.x` to `5.x` etc.
	2. Debian itself releases an update.
2.  The Runtime image is `alpine:3.10` and contains `dotnet/core/aspnet:3.0`.  This will upgraded when:
	1.  There's a update to the runtime `3.x` to `5.x` etc.
	2. Alpine itself gets updated.
3. The application itself.  This should be updated:
	1. When the SDK & runtime to get updated to include security fixes in the toolchain and to get latest platform fixes
	2. When new features or fixes are added to the app itself.

**That's 6 update cadence points!**

> You should plan on a monthly update schedule to cover OS updates and **should be comfortable kicking off an ad hoc build at any time** to cover security fixes from the libraries the app uses.

> **The build pipeline is the heart of your project.**  The pipeline should run every times a change to the source code is pushed.  That covers application and manual dependency updates.  **It should build every night**, which makes sure you always have a potentially releasable image built on the latest SDK, application platform, and operating system updates.

```powershell
# deploy the multiple compose files
docker stack deploy -c .\numbers\docker-compose.yml -c .\numbers\prod.yml numbers

# show the services in the stack
docker stack services numbers
```
<pre>
PS ch14\exercises> docker stack deploy -c .\numbers\docker-compose.yml -c .\numbers\prod.yml numbers
Creating network numbers-prod
Creating service numbers_numbers-web
Creating service numbers_numbers-api

PS ch14\exercises> docker stack services numbers
ID             NAME                  MODE         REPLICAS   IMAGE                            PORTS
47njwi2kq2bk   numbers_numbers-api   replicated   6/6        diamol/ch08-numbers-api:latest
s3iobn7e3ep8   numbers_numbers-web   global       1/1        diamol/ch08-numbers-web:latest
</pre>

Notice the `numbers_numbers-api` is `replicated` mode while `numbers_numbers-web` is in `global` mode.
Excerpt of `prod.yml` showing global configuration
```yaml
  numbers-web:
    ports:
      - target: 80
        published: 80
        mode: host
    deploy:
      mode: global
```

- `mode: global` -- This setting in the `deploy` section configures the deployment to run one container on every node in the Swarm. The number of replicas will equal the number of nodes, and if any nodes join, they will also run a container for the service.
- `mode: host` -- This setting in the `ports` section configures the service to bind directly to port 80 on the host, and not use the ingress network. This can be a useful pattern if your web apps are lightweight enough that you only need one replica per node, but network performance is critical so you don’t want the overhead of routing in the ingress network.

This version of the numbers app still contains the broken API from [[08 Health and Dependency Checks]].  This deploys version 2 which contain health checks

There's a bug in these exercises, remove line 6 from `prod-healthcheck.yml` the `healthcheck` `test` line.

```powershell
#update the stack:
docker stack deploy -c .\numbers\docker-compose.yml -c .\numbers\prod.yml -c .\numbers\prod-healthcheck.yml -c .\numbers\v2.yml numbers

# check the statck's replica's
docker stack ps numbers
```

<pre>
PS ch14\exercises> docker stack deploy -c .\numbers\docker-compose.yml -c .\numbers\prod.yml -c .\numbers\prod-healthcheck.yml -c .\numbers\v2.yml numbers
Creating network numbers-prod
Creating service numbers_numbers-api
Creating service numbers_numbers-web
PS ch14\exercises> docker stack ps numbers
ID             NAME                                            IMAGE                        NODE             DESIRED STATE   CURRENT STATE                    ERROR     PORTS
qiq26os65x27   numbers_numbers-api.1                           diamol/ch08-numbers-api:v2   docker-desktop   Running         Running less than a second ago
ijlfm34zs3fq   numbers_numbers-api.2                           diamol/ch08-numbers-api:v2   docker-desktop   Running         Running 1 second ago
r49u8hbm8abl   numbers_numbers-api.3                           diamol/ch08-numbers-api:v2   docker-desktop   Running         Running less than a second ago
6q7svojlhsil   numbers_numbers-api.4                           diamol/ch08-numbers-api:v2   docker-desktop   Running         Running less than a second ago
esy8ngol0isj   numbers_numbers-api.5                           diamol/ch08-numbers-api:v2   docker-desktop   Running         Running less than a second ago
s3onz78san8r   numbers_numbers-api.6                           diamol/ch08-numbers-api:v2   docker-desktop   Running         Running less than a second ago
hf2pz8omcchp   numbers_numbers-web.dvtq62nzdhh8imxd17na64slp   diamol/ch08-numbers-web:v2   docker-desktop   Running         Running 1 second ago                       *:80->80/tcp
</pre>

As you run the app, [http://localhost](http://localhost) the API will start to break, but Swarm will restart the containers to keep the app healthy.  Service updates are done in stages and old containers have to stop before new ones are spun up.

> The services will be under capacity while old containers are shut down and replacements are starting up.

Swarm is cautious with it's rollout of updates, one replica at a time and pauses if the new container doesn't start correctly.  However, this strategy can lead to a half-rolled out system with pieces broken.  Rollouts can be configured.

## 14.2 - Configuring production rollouts with Compose

```yaml
services:
  numbers-api:
    deploy:
      update_config:
        parallelism: 3
        monitor: 60s
        failure_action: rollback
        order: start-first
```

The four properties of the update configuration section change how the rollout works:

- `parallelism` is the number of replicas that are replaced in parallel. The default is 1, so updates roll out by one container at a time. The setting shown here will update three containers at a time. That gives you a faster rollout and a greater chance of finding failures, because there are more of the new replicas running.
- `monitor` is the time period the Swarm should wait to monitor new replicas before continuing with the rollout. The default is 0, and you definitely want to change that if your images have health checks, because the Swarm will monitor health checks for this amount of time. This increases confidence in the rollout.
- `failure_action` is the action to take if the rollout fails because containers don’t start or fail health checks within the `monitor` period. The default is to pause the rollout; I’ve set it here to automatically roll back to the previous version.
- `order` is the order of replacing replicas. `stop-first` is the default, and it ensures there are never more replicas running than the required number, but if your app can work with extra replicas, `start-first` is better because new replicas are created and checked before the old ones are removed.

> There’s one important thing to understand: when you deploy changes to a stack, the update configuration gets applied first. Then, if your deployment also includes service updates, the rollout will happen using the new update configuration.

After a few updates, `docker stack ps <stack>` can get a little difficult to read.
<pre>
ID             NAME                                                IMAGE                        NODE             DESIRED STATE
7ko9vyimyvue   numbers_numbers-api.1                               diamol/ch08-numbers-api:v3   docker-desktop   Running      
y3l82o75cxp5    \_ numbers_numbers-api.1                           diamol/ch08-numbers-api:v2   docker-desktop   Shutdown     
0e6hiznvfnr0   numbers_numbers-api.2                               diamol/ch08-numbers-api:v3   docker-desktop   Running      
qc6z884rsmli    \_ numbers_numbers-api.2                           diamol/ch08-numbers-api:v2   docker-desktop   Shutdown     
x8kmcwtfajs7   numbers_numbers-api.3                               diamol/ch08-numbers-api:v3   docker-desktop   Running      
xxky8cwaml4q    \_ numbers_numbers-api.3                           diamol/ch08-numbers-api:v2   docker-desktop   Shutdown     
nt1hxci1qo0e   numbers_numbers-api.4                               diamol/ch08-numbers-api:v3   docker-desktop   Running      
yls720k3e6vu    \_ numbers_numbers-api.4                           diamol/ch08-numbers-api:v2   docker-desktop   Shutdown     
ugq2ex5som5y   numbers_numbers-api.5                               diamol/ch08-numbers-api:v3   docker-desktop   Running      
bpvdkc7tfybd    \_ numbers_numbers-api.5                           diamol/ch08-numbers-api:v2   docker-desktop   Shutdown     
k0a3hpkm6vk7   numbers_numbers-api.6                               diamol/ch08-numbers-api:v3   docker-desktop   Running      
v7v8raiey7ap    \_ numbers_numbers-api.6                           diamol/ch08-numbers-api:v2   docker-desktop   Shutdown     
kzy71sg5cv4a   numbers_numbers-web.dvtq62nzdhh8imxd17na64slp       diamol/ch08-numbers-web:v3   docker-desktop   Ready        
5m40ohgel4k1    \_ numbers_numbers-web.dvtq62nzdhh8imxd17na64slp   diamol/ch08-numbers-web:v2   docker-desktop   Shutdown     
</pre>

Use `docker service inspect --pretty numbers_numbers-api` to get more details about the services in the stack.
<pre>
ID:             gogi8wks8z67no8bzyk9w05mp
Name:           numbers_numbers-api
Labels:
 com.docker.stack.image=diamol/ch08-numbers-api:v3
 com.docker.stack.namespace=numbers
Service Mode:   Replicated
 Replicas:      6
UpdateStatus:
 State:         completed
 Started:       6 minutes ago
 Completed:     5 minutes ago
 Message:       update completed
Placement:
UpdateConfig:
 Parallelism:   3
 On failure:    rollback
 Monitoring Period: 1m0s
 Max failure ratio: 0
 Update order:      start-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         diamol/ch08-numbers-api:v3@sha256:8e01f3f3fa0cc3596a979930437dacb425b3047436513f2615a87063a45b7491
Resources:
 Limits:
  CPU:          0.5
  Memory:       75MiB
Networks: numbers-prod
Endpoint Mode:  vip
 Healthcheck:
  Interval = 2s
  Retries = 2
  StartPeriod = 5s
  Timeout =     3s
</pre>

Specifically the `UpdateConfig` and `RollbackConfig` sections for how the service is configured to handle updates and rollbacks.

## 14.3 - Configuring service rollbacks
Configuring rollbacks are identical to configuring updates.

```powershell
# create a failed deployment
PS ch14\exercises> docker stack deploy -c ./numbers/docker-compose.yml -c ./numbers/prod.yml -c ./numbers/prod-healthcheck.yml -c ./numbers/prod-update-config.yml -c ./numbers/v5-bad.yml numbers
```
<pre>
ID:             gogi8wks8z67no8bzyk9w05mp
Name:           numbers_numbers-api
<span style="color: blue">Labels:
 com.docker.stack.image=diamol/ch08-numbers-api:v3
 com.docker.stack.namespace=numbers</span>
Service Mode:   Replicated
 Replicas:      6
<span style="color: red">UpdateStatus:
 State:         rollback_completed
 Started:       41 seconds ago
 Message:       rollback completed</span>
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
</pre>

Even though `v5` was deployed, `v3` is running, to show the service has been rolled back.  It uses the default rollback strategy. 
 This config fails faster to limit downtime during the rollback, but using `start-first` and upping the `parallelism`.
```yaml
services:
  numbers-api:
    deploy:
      rollback_config:
        parallelism: 6
        monitor: 0s
        failure_action: continue
        order: start-first
```
This is an aggressive rollback policy that assumes the previous version was good and will become good again when the replicas start.

## 14.4 - Managing downtime for your cluster

While orchestrators can deal with container issues, the machines that run the clusters can also have possible down time.

[Play with Docker](https://labs.play-with-docker.com/)

To take a node down for something like an OS update, you use...
`docker node update --availability drain <nodeid>`

Drain mode stops all replicas and prevents more from being scheduled on the node.  Manager nodes have a few differences:
- They are still in the management group
- They provide access to the management API
- Can still be elected leader

Multiple managers are necessary for high availability, but only one manager, the leader, is controlling the cluster.  The other managers have a copy of the cluster database and can take over if the leader fails, but they aren't in control.  For a new leader to be elected requires a majority vote and that means an **odd number of managers are required**.  A worker can be promoted to a manager if an even number of managers situation arises.
`docker node promote <nodeid>`

Uncommon scenarios (but still important)
- All managers go offline
	- Everything still works except monitoring.  Nothing is monitoring the services which means if a container fails nothing can spin up a replacement
	- Solution: bring new managers online
- Leader and all but one manager go offline
	- Again everything still works, but with only one manager a new leader can't be elected.
	- Solution: run `swam init` on the manager with `force-new-cluster` argument.  This manager will become the leader and the cluster data and tasks will all be preserved.
- Rebalancing replications for even distribution
	- New nodes don't run replicas by default.  The nodes will sit idle until they are rebalanced.
	- Solution: run `service update --force` to rebalance.

## 14.5 - Understanding high availability in Swarm clusters

There are multiple layers in the app deployment where HA needs to be considered.
- Health checks - tells the cluster your app is working and will keep it online if containers fail.
- Multiple worker nodes - extra capacity if a node goes offline
- Multiple managers - provide redundancy for scheduling containers and monitoring workers.

How do deal with datacenter redundancy?  The only safe way is multiple clusters.  Nodes in swarm are chatty and trying to span across data centers would bog down the system with too much latency.

User [[Domain Name System|DNS]] to direct users to the closest cluster if both are online.  Script the configuration so all clusters have the same setup.
