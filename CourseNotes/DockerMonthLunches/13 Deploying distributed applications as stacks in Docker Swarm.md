---
tags: [docker/orchestration,container/orchestration,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Deploying distributed applications as stacks in Docker Swarm

In production environments, you'll be using [[YAML]] file to describe the applications and deploy it as stacks.

Docker Swarm and Kubernetes both use this [[desired-state]] approach, but have different syntaxes.

## 13.1 - Using Docker Compose for production deployments

Docker Swarm gets an advantage since it uses a similar approach to Docker Compose.  The very simplest deployment for a Swarm is identical to a simple Compose file

```yaml
version: "3.7"
services:
  todo-web:
    image: diamol/ch06-todo-list
    ports:
      - 8080:80
```

Simple Compose file as a stack
```powershell
cd ch13/exercises

# deploy the stack from the Compose file:
docker stack deploy -c ./todo-list/v1.yml todo

# list all the stacks and see the new one:
docker stack ls

# list all services and see the service created by the deployment:
docker service ls
```
<pre>
PS ch13\exercises> docker stack deploy -c .\todo-list\v1.yml todo
Creating network todo_default
Creating service todo_todo-web
PS ch13\exercises> docker stack ls
NAME      SERVICES   ORCHESTRATOR
todo      1          Swarm
PS ch13\exercises> docker service ls
ID             NAME            MODE         REPLICAS   IMAGE                          PORTS
7njatd4ekzye   todo_todo-web   replicated   1/1        diamol/ch06-todo-list:latest   *:8080->80/tcp
PS ch13\exercises>
</pre>

The Compose file was sent to the cluster, and the manager created a default network and a service.  Stacks are first-class resources in Swarm mode and has it's own CLI commands.

A standard Docker Compose file with no extra config was deployed to the Swarm.  If multiple nodes existed, it would have high availability.

```powershell
# list all the services in the stack:
docker stack services todo

# list all replicas for all services in the stack:
docker stack ps todo

# remove the stack:
docker stack rm todo
```

## 13.2 - Managing app configuration with config objects
Apps running in containers need to be able to load their configuration settings from the platform that is running the container.  In lower environments, Docker Compose used environment variables.  In production, Docker config objects stored on the cluster are used.

> It's the exact same Docker image in every environment.  It's just the application behavior changes.

```powershell
# create the config object from a local JSON file:
docker config create todo-list-config ./todo-list/configs/config.json

# check the configs in the cluster:
docker config ls
```

Config objects can store any time of data, XML, key/value pairs, even binary files.  The Swarm delivers the config object as a file in the container's filesystem, so the application sees the *exact* same data that was uploaded.

Using `docker inspect config --pretty <config id>` the contents of the config object are displayed. 

> Config objects are **NOT SECURE**.  Anyone with access to the cluster can see the contents.

Config objects referenced in YAML file.
```yaml
version: "3.7"

services:
  todo-web:
    image: diamol/ch06-todo-list
    ports:
      - 8080:80
    configs:
      - source: todo-list-config
        target: /app/config/config.json
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: "0.50"
          memory: 100M
    networks:
      - app-net

  todo-db:
    image: diamol/postgres:11.5
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: "0.50"
          memory: 500M
    networks:
      - app-net

configs:
  todo-list-config:
    external: true

networks:
  app-net:
```

`External` is how you specify that this resource should already exist on the cluster. 

Sensitive data shouldn't be kept in config objects, because they're not encrypted and they can be read by anyone who has access to the cluster. That includes database connection strings that might have usernames and passwords, and also URLs for production services and API keys. You should aim for defense in depth in your production environment, so even if the chances of someone gaining access to your cluster are slim, you should still encrypt sensitive data inside the cluster. Docker Swarm provides secrets for storing this class of config.

## 13.3 - Managing confidential settings with secrets

These are nearly identical to config objects, except they are encrypted in the cluster database.  You can only read them in plain text inside the container when they are loaded from the Swarm.

Secrets are encrypted throughout their lifetime in the cluster.

```powershell
# create the secret from a local JSON file:
docker secret create todo-list-secret ./todo-list/secrets/secrets.json

# inspect the secret with the pretty flag to see the data:
docker secret inspect --pretty todo-list-secret
```

Secrets setup in YAML file
```yaml
version: "3.7"

services:
  todo-web:
    image: diamol/ch06-todo-list
    ports:
      - 8080:80
    configs:
      - source: todo-list-config
        target: /app/config/config.json
    secrets:
      - source: todo-list-secret
        target: /app/config/secrets.json
#...

secrets:
  todo-list-secret:
    external: true
```

## 13.4 Storing data with volumes in the Swarm

Volumes also work similarly to how they do with Docker Compose.  However, there is one big difference.  In a cluster you'll have multiple nodes that can run containers, and each node has it's own disk where it stores local volumes.  The simplest way to main states between updates is to use a local volume.

Here's the problem, a replacement replica might be scheduled to run on a different node than the original.  It will not have access to the original nodes data.

You can pin services to a specific node, by labelling the node and specifying the label in the Compose file.

> That works for scenarios where you want application data to be stored outside of the container so it survives updates, but where you don't need to run multiple replicas and you don't need to allow for server failure.

Which to me, completely defeats the point :-)
```powershell
# find the ID for your node and update it, adding a label:
docker node update --label-add storage=raid $(docker node ls -q)
```

Specifying node in Compose file
```yaml
version: "3.7"

services:
  todo-db:
    image: diamol/postgres:11.5
    environment:
      PGDATA: "/var/lib/postgresql/data/pgdata"
    volumes:
      - todo-db-data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: "0.50"
          memory: 500M
      placement:
        constraints:
          - node.labels.storage == raid
# ...

volumes:
  todo-db-data:
```

> That volume has the same lifetime as the stack, so if you remove the stack, the volumes get removed, and if you update services, they'll get a new default volume. If you want your data to persist between updates, you need to use a named volume in your Compose file.

Docker has a plugin system for volume drivers, so Swarms can be configured to provide distributed storage using a cloud storage system or a storage device in the datacenter. Configuring those volumes depends on the infrastructure you're using, but you consume them in the same way, attaching volumes to services.

## 13.5 - Understanding how the cluster manages stacks

Stacks are groups of resources that the cluster managers for you.  

- Volumes - A default value will be created if the image specifies one and will be removed when the stack is removed.  Use named volumes if you want data to be persisted.
- Secrets and configs - Uploaded to the cluster, write-once read-everywhere objects and are immutable once uploaded.
- Networks - Can be managed externally or by the the Swarm which will create and remove them automatically.  Every Stack will be deployed with a network, even if not specified in the Compose file.
- Services - Created or removed when the stack is deployed.  Constantly monitored by the Swarm and replaced when health checks fail.

Unlike Docker Compose, Stacks can't control deployment dependency order (a la `depends on`).  Assume components start in random order and use health and dependency checks in the image so the containers fail fast.