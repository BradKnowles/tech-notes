---
tags: [docker,diamol]
---

source: [[Docker in a Month of Lunches]]

# Packaging applications from source code into Docker Images

> There’s one other thing you need to know to package your own applications: you can also run commands inside Dockerfiles.

Commands execute during build and changes are saved in the image.

## 4.1 - Who needs a build server when you have a Dockerfile?

Multi-stage Dockerfile have multiple stages to the build.  Each stages starts with `FROM` and you can name them with the `AS` parameter.  They run independently, but things can be copied between stages.

The `RUN` instructions can be used to write files.  It executes commands during the build and saves the output in the image layer.

> It’s important to understand that the individual stages are isolated. You can use different base images with different sets of tools installed and run whatever commands you like. The output in the final stage will only contain what you explicitly copy from earlier stages. If a command fails in any stage, the whole build fails.

```powershell
cd ch04/exercises/multi-stage
docker image build -t multi-stage .
 ```

 <pre>
[+] Building 0.9s (9/9) FINISHED
 => [internal] load build definition from Dockerfile                                              0.0s
 => => transferring dockerfile: 311B                                                              0.0s
 => [internal] load .dockerignore                                                                 0.0s
 => => transferring context: 2B                                                                   0.0s
 => [internal] load metadata for docker.io/diamol/base:latest                                     0.0s
 => <strong>[stage-2 1/2]</strong> FROM docker.io/diamol/base                                                      0.0s
 => [<strong>build-stage 2/2]</strong> RUN echo 'Building...' > /build.txt                                         0.3s
 => [<strong>test-stage 2/3]</strong> COPY --from=build-stage /build.txt /build.txt                                0.0s
 => [<strong>test-stage 3/3]</strong> RUN echo 'Testing...' >> /build.txt                                          0.5s
 => [<strong>stage-2 2/2]</strong> COPY --from=test-stage /build.txt /build.txt                                    0.0s
 => exporting to image                                                                            0.0s
 => => exporting layers                                                                           0.0s
 => => writing image sha256:d79dc081523ef748bded5476cda94076454ed0806837d4e530baab9ad55049a9      0.0s
 => => naming to docker.io/library/multi-stage                                                    0.0s
 </pre>

A multi-stage Dockerfile for a Java application.
 ![[MultistageJavaDockerImage.png]]

## 4.2 - App walkthrough: Java source code

Example Java application from [ch04/exercises/image-of-the-day](https://github.com/sixeyed/diamol/tree/master/ch04/exercises/image-of-the-day)

```dockerfile
FROM diamol/maven AS builder

WORKDIR /usr/src/iotd
COPY pom.xml .
RUN mvn -B dependency:go-offline

COPY . .
RUN mvn package

# app
FROM diamol/openjdk

WORKDIR /app
COPY --from=builder /usr/src/iotd/target/iotd-service-0.1.0.jar .

EXPOSE 80
ENTRYPOINT ["java", "-jar", "/app/iotd-service-0.1.0.jar"]

```

- Two stages `builder` and unamed
- `WORKDIR` creates the directory and navigating to it
- `RUN mvn -B dependency:go-offline` gets it's own `RUN` instruction because it's an expensive operation and we can take advantage of Docker's caching layer.  If no dependencies change between builds, the cache layer will be used.
- `COPY . .` means "copy all files and directories from the location where the Docker build is running, into the working directory in the image"
- `RUN MVN package` compiles and packages the application

The compiled application now exists in the `builder` stage's filesystem
- `FROM diamol/openjdk` starts a new stage where none of the builder tools exist, just the runtime.
- `COPY --from=builder /usr/src/iotd/target/iotd-service-0.1.0.jar` copies the jar file from the builder to the /app directory defined above.
- `EXPOSE` port 80 so the app can receive requests
- `ENTRYPOINT` when the container is run from an image, run the app

### Networks
When using multiple container, they can communicate on virtual networks.  When you run containers you can explicitly connect them to that Docker network using the --network flag, and any containers on that network can reach each other using the container names.

```powershell
docker network create nat

docker container run --name iotd -d -p 800:80 --network nat image-of-the-day
```

[http://localhost:800/image](http://localhost:800/image)
```json
{
  "url": "https://apod.nasa.gov/apod/image/2203/multiWcrab_lg1024c.jpg",
  "caption": "The Multiwavelength Crab",
  "copyright": null
}
```

### Note on final state contents
> One other thing to be really clear on here: the build tools are not part of the final application image. You can run an interactive container from your new `image-of-the-day` Docker image, and you’ll find there's no `mvn` command in there. Only the contents of the final stage in the Dockerfile get made into the application image; anything you want from previous stages needs to be explicitly copied in that final stage.

## 4.3 - App walkthrough: Node.js source code

Example Node.js app from [/ch04/exercises/access-log](https://github.com/sixeyed/diamol/tree/master/ch04/exercises/access-log)

```dockerfile
FROM diamol/node AS builder

WORKDIR /src
COPY src/package.json .
RUN npm install

# app

FROM diamol/node

EXPOSE 80
CMD ["node", "server.js"]

WORKDIR /app

COPY --from=builder /src/node_modules/ /app/node_modules/

COPY src/ .
```

```powershell
cd ch04/exercises/access-log
docker image build -t access-log .

docker container run --name accesslog -d -p 801:80 --network nat access-log # Connects to nat network
```

[http://localhost:801/stats](http://localhost:801/stats)

## 4.4 - App walkthrough: Go source code
```dockerfile
FROM diamol/golang AS builder
COPY main.go .

RUN go build -o /server

# app
FROM diamol/base

ENV IMAGE_API_URL="http://iotd/image" \
      ACCESS_API_URL="http://accesslog/access-log"
CMD ["/web/server"]

WORKDIR web
COPY index.html .
COPY --from=builder /server .
RUN chmod +x server
```

```powershell
cd ch04/exercises/image-gallery
docker image build -t image-gallery .

```

Compare image sizes
```powershell
docker image ls -f reference=diamol/golang -f reference=image-gallery
```

<pre>
REPOSITORY      TAG       IMAGE ID       CREATED         SIZE
image-gallery   latest    4d042ef0c7d0   2 minutes ago   27.1MB
diamol/golang   latest    119cb20c3f56   18 months ago   803MB
</pre>

By using a minimal base image for the application, we’re saving nearly *775 MB* of software, which is a huge reduction in the surface area for potential attacks.

```powershell
docker container run -d -p 802:80 --network nat image-gallery
```
[http://localhost:802/](http://localhost:802/)

Right now you’re running a distributed application across three containers. The Go web application calls the Java API to get details of the image to show, and then it calls the Node.js API to log that the site has been accessed. You didn’t need to install any tools for any of those languages to build and run all the apps; you just needed the source code and Docker.

## 4.5 - Understanding multi-stage Dockerfiles
- Standardization - It doesn’t matter what operating system you have or what’s installed on your machine--all the builds run in Docker containers, and the container images have all the correct versions of the tools.
- Performance - Using cache is important. Spend time organizing Dockerfiles to optimize use of cache.
- Finely-tuned builds - The goal is a final image that is as lean as possible. No build tools or other apps that are unnecessary, just your app.
