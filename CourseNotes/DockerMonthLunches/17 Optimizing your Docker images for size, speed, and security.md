---
tags: [docker/images,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Optimizing your Docker images for size, speed, and security

Deploying to production is more than just creating images.  There are ways to decrease image size, lessen deployment times, and increase the security of the images.

## 17.1 - How you optimize Docker images

Layers that are shared between images don't automatically get removed when containers are replaced.

Shows how much disk space Docker is using
```powershell
docker system df
```
<pre>
PS > docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          96        9         5.43GB    4.98GB (91%)
Containers      15        0         141.9MB   141.9MB (100%)
Local Volumes   9         3         939.7MB   596.1MB (63%)
Build Cache     242       0         526.5MB   526.5MB
</pre>

Clear out image layers and build cache without removing full images
```powershell
docker system prune
```

### Optimizing 
#### Don't include files in the image unless you need them.

```Docker
# Dockerfile v1 - copies in the whole directory structure:
FROM diamol/base
CMD echo app- && ls app && echo docs- && ls docs
COPY . .

# Dockerfile v2 - adds a new step to delete unused files
FROM diamol/base
CMD echo app- && ls app && echo docs- && ls docs
COPY . .
RUN rm -rf docs
```

Dockerfile v2 tries to remove the extra files copied.  This doesn't work.  The files are just hidden in the delete layer, but still exist.

Each instruction in a Dockerfile creates an image layer and all layers are merged together to form the image.  Writing files to a layer permanently writes the files, deleting them in a later layer just hides them in the filesystem.

```Docker
FROM diamol/base
CMD echo app- && ls app && echo docs- && ls docs
COPY ./app ./app
```

This is optimized because it only `COPY`s what's necessary.

### Use Dockerignore to reduce the size of the Docker context.

Docker sends the build context, the directory where the build is run, and sends it to the engine.  Using `.dockerignore` excludes unnecessary files from the context.

```Docker
.git
docs/
Dockerfile*
```

## 17.2 - Choosing the right base image

It's important to choose the right image to limit security exposure and keep image sizes small.

 As a good rule, use Alpine or Debian Slim images as the base OS for Linux containers, and Nano Server for Windows container.

Only build the image with the tools necessary to run the application.  Developer tools for building can be a security risk.

Use golden images where possible.  A golden image is a base image created and managed by your or your organization.  These image can be upgraded on the organizations schedule.

## 17.3 Minimizing image layer count and layer size

### Multiple commands single `RUN`

```Docker
# Dockerfile - the naive install with APT:
FROM debian:stretch-slim
RUN apt-get update
RUN apt-get install -y curl=7.52.1-5+deb9u9
RUN apt-get install -y socat=1.7.3.1-2+deb9u1
```

```Docker
# Dockerfile.v2 - optimizing the install steps:
FROM debian:stretch-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
       curl=7.52.1-5+deb9u9 \
       socat=1.7.3.1-2+deb9u1 \
 && rm -rf /var/lib/apt/lists/*
```

The difference is that multiple command are issues with a single `RUN` instruction and includes an optimization to delete the package lists to save more disk space.

Optimized method for downloading and extracting files
```Docker
FROM diamol/base
 
ARG DATASET_URL=https://archive.ics.uci.edu/.../url_svmlight.tar.gz

WORKDIR /dataset

RUN wget -O dataset.tar.gz ${DATASET_URL} && \
      tar -xf dataset.tar.gz url_svmlight/Day1.svm && \
        rm -f dataset.tar.gz
```
The optimized part is extracting the single file that is needed and from deleting the archive file.

### Multi-stage Docker files

The ultimate optimization is using multi-stage builds to abstract the disk hungry steps.

```Docker
FROM diamol/base AS download
ARG DATASET_URL=https://archive.ics.uci.edu/.../url_svmlight.tar.gz
RUN wget -O dataset.tar.gz ${DATASET_URL}

FROM diamol/base AS expand
COPY --from=download dataset.tar.gz .
RUN tar xvzf dataset.tar.gz

FROM diamol/base
WORKDIR /dataset/url_svmlight
COPY --from=expand url_svmlight/Day1.svm .
```
The resulting image is optimized since the last image only contains what is necessary and it's easier to debug.

### Optimizing build cache
Ordering instructions so the things that change the least frequently are at the start and the the things that change most frequently are at the bottom is the basic way to optimize the build cache.