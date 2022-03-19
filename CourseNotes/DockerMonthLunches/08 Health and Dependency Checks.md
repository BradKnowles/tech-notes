---
tags: [docker/health,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Supporting reliability with health checks and dependency checks

Running apps in production with things like [[Docker Swarm]] or [[Kubernetes]] helps with self-healing apps.  You can package your containers with information the platform uses to check if the application inside the container is healthy. If the app stops working correctly, the platform can remove a malfunctioning container and replace it with a new one.

## 8.1 - Building health checks into Docker images

Docker contains very basic health checks.  If the app process stops running, the container exits.  Docker can restart those, but doesn't know about web apps that are returning 500s yet the process is still running.

Run a container with an app that will start failing
```powershell
# start the API container
docker container run -d -p 8080:80 diamol/ch08-numbers-api

# repeat this three times - it returns a random number
curl http://localhost:8080/rng
curl http://localhost:8080/rng
curl http://localhost:8080/rng

# from the fourth call onwards, the API always fails
curl http://localhost:8080/rng

# check the container status
docker container ls
```

<pre>
PS> curl http://localhost:8080/rng
95
PS> curl http://localhost:8080/rng
66
PS> curl http://localhost:8080/rng
47
PS> curl http://localhost:8080/rng
{"type":"https://tools.ietf.org/html/rfc7231#section-6.6.1","title":"An error occured while processing your request.","status":500,"traceId":"|faa9f237-421be707f28f44d1."}
</pre>

The `HEALTHCHECK` instruction
```Dockerfile
FROM diamol/dotnet-aspnet

ENTRYPOINT ["dotnet", "/app/Numbers.Api.dll"]
HEALTHCHECK CMD curl --fail http://localhost/health

WORKDIR /app
COPY --from=builder /out/ .
```

This calls the `health` endpoint on the API and with the `--fail` flag any errors received by `curl` will be passed to Docker, thus marking the container as `Unhealthy`.

The `HEALTHCHECK` is configurable, but by default it's every 30 seconds and 3 failures in a row.

```powershell
# start the API container, v2
docker container run -d -p 8081:80 diamol/ch08-numbers-api:v2

# wait 30 seconds or so and list the containers
docker container ls

# repeat this four times - it returns three random numbers and then fails
curl http://localhost:8081/rng
curl http://localhost:8081/rng
curl http://localhost:8081/rng
curl http://localhost:8081/rng

# now the app is in a failed state - wait 90 seconds and check
docker container ls
```

<pre>
PS> docker container ls
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS                    PORTS                  NAMES
2ca8c3e4e561   diamol/ch08-numbers-api:v2   "dotnet /app/Numbers…"   33 seconds ago   Up 32 seconds <mark>(healthy)</mark>   0.0.0.0:8081->80/tcp   goofy_feistel
</pre>

After causing the error...
<pre>
PS > docker container ls
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS                     PORTS                  NAMES
2ca8c3e4e561   diamol/ch08-numbers-api:v2   "dotnet /app/Numbers…"   5 minutes ago    Up 5 minutes <mark>(unhealthy)</mark>   0.0.0.0:8081->80/tcp   goofy_feistel
</pre>

> Docker doesn't stop the container because it's running a single server and doesn't really know what to do.  In a production environment [[Docker Swarm]] or [[Kubernetes]] would deal with restarting the container etc.

Using `docker container inspect` you can view the the health check logs by looking at the `{"State": {"Health":{...}}` object to see the status and most recent logs.

## 8.2 - Starting containers with dependency checks

Running in a cluster you lose the control of startup order that [[Docker Compose]] gives you.  Remember, Docker Compose is a client-side tool, not one used in production.

You can add dependency checks to make sure containers that depend on others fail fast.

There isn't a special keyword for it but you can add logic in the startup command.

Example Dockerfile
```Dockerfile
FROM diamol/dotnet-aspnet
 
ENV RngApi:Url=http://numbers-api/rng

CMD curl --fail http://numbers-api/rng && \
      dotnet Numbers.Web.dll

WORKDIR /app

COPY --from=builder /out/ .
```

In the `CMD` instruction, `curl` is called to check on the dependent api. If it succeeds, `dotnet Numbers.web.dll` is executed.  If `curl` fails, the container exits.

> It's counterintuitive, but in this scenario it's better to have an exited container than a running container. This is fail-fast behavior, and it's what you want when you're running at scale. When a container exits, the platform can schedule a new container to come up and replace it. Maybe the API container takes a long time to start up, so it's not available when the web container runs; in that case the web container exits, a replacement is scheduled, and by the time it starts the API is up and running.

## 8.3 - Writing custom utilities for application check logic

From [[04 Packaging applications from source code into Docker Images|Chapter 4]] the Docker image should have the bar minimum necessary for the application to run.  It's better to write the utility checks in the same language your application uses.

Advantages
- You reduce the software requirements in your image--you don't need to install any extra tools, because everything the check utility needs to run is already there for the application.
- You can use more complex conditional logic in your checks with retries or branches, which are harder to express in shell scripts, especially if you're publishing cross-platform Docker images for Linux and Windows.
- Your utility can use the same application configuration that your app uses, so you don't end up specifying settings like URLs in several places, with the risk of them getting out of sync.
- You can execute any tests you need, checking database connections or file paths for the existence of certificates that you're expecting the platform to load into the container--all using the same libraries your app uses.

Example of Healthcheck Dockerfile with Utility app
```dockerfile
FROM diamol/dotnet-aspnet
ENTRYPOINT ["dotnet", "Numbers.Api.dll"]
# ***
HEALTHCHECK CMD ["dotnet", "Utilities.HttpCheck.dll", "-u", "http://localhost/health"]
# ***
WORKDIR /app
COPY --from=http-check-builder /out/ .
COPY --from=builder /out/ .
```

Example of startup check
```dockerfile
FROM diamol/dotnet-aspnet

ENV RngApi:Url=http://numbers-api/rng

# ***
CMD dotnet Utilities.HttpCheck.dll -c RngApi:Url -t 900 && \
      dotnet Numbers.Web.dll
# ***

WORKDIR /app

COPY --from=http-check-builder /out/ .
COPY --from=builder /out/ .
```

This removes the `curl` requirement from the image.

## 8.4 - Defining health checks and dependency checks in Docker Compose

Docker Compose can go some of the way toward repairing unreliable applications, but it won't replace unhealthy containers for the same reasons that Docker Engine won't: you're running on a single server, and the fix might cause an outage. But it can set containers to restart if they exit, and it can add a health check if there isn't one already in the image.

Specifying health check parameters Docker Compose
```dockerfile
   numbers-web:
       image: diamol/ch08-numbers-web:v3
       restart: on-failure
       ports:
           - "8088:80"
       healthcheck:
           interval: 5s
           timeout: 1s
           retries: 2
           start_period: 10s
       networks:
            - app-net
```
|Instruction    | Description|
|---------------|------------|
|interval       | the time between checks--in this case five seconds|
|timeout        | how long the check should be allowed to run before it's considered a failure|
|retries        | number of consecutive failures allowed before the container is flagged as unhealthy|
|startup_period | amount of time to wait before triggering the health check, which lets you give your app some startup time before health checks run|

If the image doesn't have a healthcheck defined, adding a `test` property to `healthcheck` will define what to run.
`test: ["CMD", "dotnet", "Utilities.HttpCheck.dll", "-t", "150"]`

`restart: on-failure` will restart the container if it stops unexpectedly.

> You might ask, why bother building a dependency check into the container startup when Docker Compose can do it for you with the `depends_on` flag? The answer is that Compose can only manage dependencies on a single machine, and the startup behavior of your app on a production cluster is far less predictable.

## 8.5 - Understanding how checks power self-healing apps

While agility and flexibility are advantages to distributed systems, management becomes more complicated.

> There will be lots of dependencies between components, and it's tempting to want to declare the order in which components get started so you can model the dependencies. But it’s really not a great idea to do that.

On a single machine controlling startup order is easy, but in an environment with multiple servers and an orchestrator like [[Kubernetes]] that spins up 20 or 30 instances of your container, it will be difficult to spin up in the right order.

Dependency and health checks are more useful in production.  If a container can't find it's dependency, it fails fast, restarts and will eventually find the dependency and run healthy.

> The idea of self-healing apps is that any transient failures can be dealt with by the platform. If your app has a nasty bug that causes it to run out of memory, the platform will shut down the container and replace it with a new one that has a fresh allocation of memory. It doesn't fix the bug, but it keeps the app working correctly.

Health checks run periodically and shouldn't do too many things.  You don't want the health check itself impacting app performance.  It's a balance to find the right amount of checking.
