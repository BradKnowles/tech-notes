---
tags: [docker/compose,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Running multi-container apps with Docker Compose

[[Docker Compose]] is a file format and tool for describing distributed Docker apps.

## 7.1 - The anatomy of a Docker Compose file

The Docker Compose file describes the desired state of your app.  Instead of running multiple separate Dockerfiles individually, Docker compose bundles them into a single file.

Example Docker Compose file

```yaml
version: '3.7'

services:
  
  todo-web:
    image: diamol/ch06-todo-list
    ports:
      - "8020:80"
    networks:
      - app-net

networks:
  app-net:
    external:
      name: nat
```

> Spaces are import in [[YAML]] and indentation is used to identify objects and the child properties of objects.

This YAML file has three top level statements
- `version` - The version of the Docker Compose format used in this file.
- `services` - Lists all the components that make up the application.
- `networks` - All the Docker networks that the services containers can plug into.

This Docker Compose file is the same as running this from the CLI.
```powershell
docker container run -p 8020:80 --name todo-web --network nat diamol/ch06-todo-list
```

The service name becomes the container name and the DNS name of the container.

The network name in the service is `app-net`, but under the networks section that network is specified as mapping to an external network called `nat` . The external option means Compose expects the `nat` network to already exist, and it won’t try to create it.

```powershell
docker network create nat

cd ./ch07/exercises/todo-list

docker-compose up
```

Docker compose expects to find a file called `docker-compose.y[a]ml` in the current directory.

<pre>
[+] Running 1/0
 - Container todo-list-todo-web-1  Created                                                                         0.1s
Attaching to todo-list-todo-web-1
todo-list-todo-web-1  | warn: Microsoft.AspNetCore.DataProtection.Repositories.FileSystemXmlRepository[60]
todo-list-todo-web-1  |       Storing keys in a directory '/root/.aspnet/DataProtection-Keys' that may not be persisted outside of the container. Protected data will be unavailable when container is destroyed.
todo-list-todo-web-1  | warn: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[35]
todo-list-todo-web-1  |       No XML encryptor configured. Key {dad040bc-c5da-496e-8b06-cbd8b1b3f0d8} may be persisted to storage in unencrypted form.
todo-list-todo-web-1  | info: Microsoft.Hosting.Lifetime[0]
todo-list-todo-web-1  |       Now listening on: http://[::]:80
todo-list-todo-web-1  | info: Microsoft.Hosting.Lifetime[0]
todo-list-todo-web-1  |       Application started. Press Ctrl+C to shut down.
todo-list-todo-web-1  | info: Microsoft.Hosting.Lifetime[0]
todo-list-todo-web-1  |       Hosting environment: Production
todo-list-todo-web-1  | info: Microsoft.Hosting.Lifetime[0]
todo-list-todo-web-1  |       Content root path: /app
</pre>

All of the logs are collected and grouped by container.

The Docker Compose file will live in source control and becomes the single place to describe all the runtime properties of the app.  It can also record other top-level Docker resources like volumes and secrets.

## 7.2 - Running a multi-container application with Compose

Multi-container Docker Compose example
```yaml
 accesslog:
   image: diamol/ch04-access-log
 iotd:
   image: diamol/ch04-image-of-the-day
   ports:
       - "80"
 
 image-gallery:
   image: diamol/ch04-image-gallery
   ports:
       - "8010:80"
 
   depends_on:
       - accesslog
       - iotd
```
- `accesslog` doesn't publish any ports or use other properties captured from `docker container run` so we only need image name
- `iotd` is the REST API so it publishes port `80` to a random port on the host.
- `image-gallery` has the image name and mapped host port `8010` to container port `80`.  The `depends_on` section lists the other services as dependencies.  Compose will make sure they are running first.

You can use [[Docker Compose Visualizer]] to generate diagrams of the Compose file every time they change.

You can also manage the application as a whole using the Compose file. The API service is effectively stateless, so you can scale it up to run on multiple containers. When the web container requests data from the API, Docker will share those requests across the running API containers.

```powershell
docker-compose up -d --scale iotd=3

# browse to http://localhost:8010 and refresh

docker-compose logs --tail=1 iotd

```

Two new containers are created to run the image API service so it now has a scale of three.

Docker Compose can show you all log entries for all containers, or you can use it to filter the output--the `--tail=1` parameter just fetches the last log entry from each of the `iotd` service containers.
<pre>
image-of-the-day-iotd-3  | 2022-03-05 23:39:08.505  INFO 1 --- [p-nio-80-exec-1] iotd.ImageController
  : Fetched new APOD image from NASA
image-of-the-day-iotd-1  | 2022-03-05 23:42:25.606  INFO 1 --- [p-nio-80-exec-1] iotd.ImageController
  : Fetched new APOD image from NASA
image-of-the-day-iotd-2  | 2022-03-05 23:39:04.186  INFO 1 --- [           main] iotd.Application
  : Started Application in 2.238 seconds (JVM running for 2.578)
</pre>

> Docker Compose is a client-side tool.  It sends commands to the Docker API based on the contents of the Docker Compose YAML file.  Docker just runs the containers, it doesn't know that all of these containers represent a single application.

It's possible to get your application out of sync with the Compose file.  Such as the `scale` command earlier.  Taking the app down and restarting it will not spin up 3 intances of the API anymore because it's not documented in the Compose file.
```powershell
docker-compose down # Stop and remove all the containers
 
docker-compose up -d
 
docker container ls
```

>The `down` command also removes networks and volumes recoded in the Compose file and not flagged as `external`.

## 7.3 - How Docker plugs containers together
How do the components in Docker Compose communicate with each other?

It's [[Domain Name System|DNS]], it's always DNS.

Docker has it's own DNS service built it.  It returns IP address for Container names and passes other requests onto the server where Docker is running.

If multiple IP addresses are returned, like when multiple container instances are running, the DNS service changes the order in which they are returned to offer a basic version of [[load balancing]].

## 7.4 - Application configuration in Docker Compose

```yaml
version: "3.7"

services:
  todo-db:
    image: diamol/postgres:11.5
    ports:
      - "5433:5432"
    networks:
      - app-net

  todo-web:
    image: diamol/ch06-todo-list
    ports:
      - "8030:80"
    environment:
      - Database:Provider=Postgres
    depends_on:
      - todo-db
    networks:
      - app-net
    secrets:
      - source: postgres-connection
        target: /app/config/secrets.json

networks:
  app-net:

secrets:
  postgres-connection:
    file: ./config/secrets.json

```

- `environment` sets up environment variables that are created inside the container. When this app runs, there will be an environment variable called `Database:Provider` set inside the container, with the value `Postgres`.
- `secrets` can be read from the runtime environment and populated as files inside the container. This app will have a file at `/app/config/secrets.json` with the contents of the secret called `postgres-connection`.

Secrets are usually provided by the container platform in a clustered environment--that could be [[Docker Swarm]] or [[Kubernetes]].  One a local machine, they are read from a file.

Plugging app configuration into the Compose file lets you use the same Docker images in different ways and be explicit about the settings for each environment. You can have separate Compose files for your development and test environments, publishing different ports and triggering different features of the app. This Compose file sets up environment variables and secrets to run the to-do app in Postgres mode and provide it with the details to connect to the Postgres database.

## 7.5 - Understanding the problem Docker Compose solves

Docker Compose is great for local testing but doesn't replace orchestration systems that handle continuous monitoring and desired state management.  You won’t get high availability, load balancing, or failover on that Docker machine.
