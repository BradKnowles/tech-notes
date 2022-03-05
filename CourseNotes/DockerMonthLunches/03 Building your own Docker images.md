---
tags: [docker,diamol]
---

source: [[Docker in a Month of Lunches]]

# Building your own Docker images
[[Docker in a Month of Lunches]]

## 3.1 - Using a container image from Docker Hub

```powershell
docker image pull diamol/ch03-web-ping
```

<pre>
PS C:\Users\Brad> docker image pull <strong>diamol/ch03-web-ping</strong>
Using default tag: latest
latest: Pulling from diamol/ch03-web-ping

<em># One image is physically stored as many layers</em>
<strong>e7c96db7181b: Pull complete
bbec46749066: Pull complete
89e5cf82282d: Pull complete
5de6895db72f: Pull complete
f5cca017994f: Pull complete
78b9b6c949f8: Pull complete</strong>

Digest: sha256:2f2dce710a7f287afc2d7bbd0d68d024bab5ee37a1f658cef46c64b1a69affd2
Status: Downloaded newer image for diamol/ch03-web-ping:latest
docker.io/diamol/ch03-web-ping:latest
</pre>

Docker's default image registry is [Docker Hub](https://hub.docker.com/)

> Docker images may be packaged with a default set of configuration values for the application, but you should be able to provide different configuration settings when you run a container.

`--env` flag on `docker container run` allows passing of environment variables to container.

## 3.2 - Writing your first Dockerfile

Dockerfiles are simple scripts used to package up applications, a set of instructions whose output is a Docker image.

```dockerfile
 FROM diamol/node
 ENV TARGET="blog.sixeyed.com"
 ENV METHOD="HEAD"
 ENV INTERVAL="3000"
 WORKDIR /web-ping
 COPY app.js .
 CMD ["node", "/web-ping/app.js"]
 ```

|Instruction | Meaning |
|------------|---------|
| FROM | Starting point for the image |
| ENV  | Sets values for environment variables in `[key]="[value]"` syntax |
| WORKDIR | Creates a directory in the image and sets that to the be the current working directory, like running `mkdir && cd`|
| COPY    | Copies file or directories from local to image, syntax `[source path] [destination path]` |
| CMD     | Command to run when image starts |

## 3.3 - Building your own container image

Each image needs a name and the location for all the files to package.

```powershell
docker image build --tag web-ping .
```

`--tag` is the name for the image. The last argument is the directory where the Dockerfile and related files are.  Docker calls this directory the "context".

<pre>
PS C:\Users\Brad\Code\Tutorials\Docker\diamol\ch03\exercises\web-ping> docker image build --tag web-ping .
[+] Building 1.5s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 200B                                                                               0.0s
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => [internal] load metadata for docker.io/diamol/node:latest                                                      1.3s
 => [internal] load build context                                                                                  0.0s
 => => transferring context: 881B                                                                                  0.0s
 => [1/3] FROM docker.io/diamol/node@sha256:dfee522acebdfdd9964aa9c88ebebd03a20b6dd573908347be3ebf52ac4879c8       0.1s
 => => resolve docker.io/diamol/node@sha256:dfee522acebdfdd9964aa9c88ebebd03a20b6dd573908347be3ebf52ac4879c8       0.0s
 => => sha256:9dfa73010b19a67a86f360868c0f50fff7b5e6d4fd76d21df822c62aa784f810 5.66kB / 5.66kB                     0.0s
 => => sha256:dfee522acebdfdd9964aa9c88ebebd03a20b6dd573908347be3ebf52ac4879c8 1.41kB / 1.41kB                     0.0s
 => => sha256:f59303fb3248e5d992586c76cc83e1d3700f641cbcd7c0067bc7ad5bb2e5b489 1.16kB / 1.16kB                     0.0s
 => [2/3] WORKDIR /web-ping                                                                                        0.0s
 => [3/3] COPY app.js .                                                                                            0.0s
 => exporting to image                                                                                             0.0s
 => => exporting layers                                                                                            0.0s
 => => writing image sha256:687ef63e69f62e84b007c8a2f8f74f0960e24f6a757dbe4c2bb6ebdde7c7e4f8                       0.0s
 => => naming to docker.io/library/web-ping
 </pre>

 List images that start with "w"
 ```powershell
 docker image is 'w*'
 ```

 ## 3.4 - Understanding Docker images and image layers

Check history of image

 ```powershell
 docker image history web-ping
 ```

 <pre>
IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
687ef63e69f6   5 minutes ago   CMD [&quot;node&quot; &quot;/web-ping/app.js&quot;]                 0B        buildkit.dockerfile.v0
&lt;missing&gt;      5 minutes ago   COPY app.js . # buildkit                        846B      buildkit.dockerfile.v0
&lt;missing&gt;      5 minutes ago   WORKDIR /web-ping                               0B        buildkit.dockerfile.v0
&lt;missing&gt;      5 minutes ago   ENV INTERVAL=3000                               0B        buildkit.dockerfile.v0
&lt;missing&gt;      5 minutes ago   ENV METHOD=HEAD                                 0B        buildkit.dockerfile.v0
&lt;missing&gt;      5 minutes ago   ENV TARGET=blog.sixeyed.com                     0B        buildkit.dockerfile.v0
&lt;missing&gt;      2 years ago     /bin/sh -c #(nop)  CMD [&quot;node&quot;]                 0B
&lt;missing&gt;      2 years ago     /bin/sh -c #(nop)  ENTRYPOINT [&quot;docker-entry…   0B
&lt;missing&gt;      2 years ago     /bin/sh -c #(nop) COPY file:238737301d473041…   116B
&lt;missing&gt;      2 years ago     /bin/sh -c apk add --no-cache --virtual .bui…   5.1MB
&lt;missing&gt;      2 years ago     /bin/sh -c #(nop)  ENV YARN_VERSION=1.16.0      0B
&lt;missing&gt;      2 years ago     /bin/sh -c addgroup -g 1000 node     &amp;&amp; addu…   64.7MB
&lt;missing&gt;      2 years ago     /bin/sh -c #(nop)  ENV NODE_VERSION=10.16.0     0B
&lt;missing&gt;      2 years ago     /bin/sh -c #(nop)  CMD [&quot;/bin/sh&quot;]              0B
&lt;missing&gt;      2 years ago     /bin/sh -c #(nop) ADD file:a86aea1f3a7d68f6a…   5.53MB
</pre>

The CREATED BY commands are the Dockerfile instructions--there’s a one-to-one relationship, so each line in the Dockerfile creates an image layer.

A Docker image is a logical collection of image layers.  Layers are the files that are physically stored in the [[Docker Engine]]'s cache.

> Image layers can be shared between different images and different containers

If you have lots of containers all running Node.js apps, they will all share the same set of image layers that contain the Node.js runtime.
![[SharedImageLayers.png]]

> The Size column in `docker image ls` can be deceiving. It's the size if no other images existed on the system.  If images are on the system the disk space used is much smaller.

Show exactly how much reach disk space Docker is using

```powershell
docker system df
```

<pre>
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          6         3         1.55GB    1.492GB (96%)
Containers      3         0         138B      138B (100%)
Local Volumes   0         0         0B        0B
Build Cache     29        0         846B      846B
</pre>

> Image layers are read-only.  Otherwise a change in an image layer could affect other containers.

## 3.5 - Optimizing Dockerfiles to use the image layer cache

Every Dockerfile instruction results in an image layer.  If the instruction doesn't change between builds, Docker can use the previous layer in the cache.

Docker uses hashing to find matching layers.  If there's no match in existing image layers, Docker executes the instruction and breaks the cache.

> As soon as the cache is broken, Docker executes all the instructions that follow, even if they haven’t changed.

> Dockerfiles should be optimized by ordering instructions on how frequently they change.  Less frequent changes near the top; more frequently changing near the bottom.

> **The goal is for most builds to only need to execute the last instruction, using the cache for everything else.**

Previous dockerfile with optimizations

```dockerfile
 FROM diamol/node
 CMD ["node", "/web-ping/app.js"]
 ENV TARGET="blog.sixeyed.com" \
       METHOD="HEAD" \
       INTERVAL="3000"
 WORKDIR /web-ping
 COPY app.js .
```

## 3.6 Lab

Save container as an image
```powershell
docker container commit ch03lab ch03-lab-soln
                        ^^^^^^^ ^^^^^^^^^^^^^
		Name of changed container
								Name of new image
						        
```