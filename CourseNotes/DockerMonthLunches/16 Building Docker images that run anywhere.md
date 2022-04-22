---
tags: [docker/images/multiarch,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Building Docker images that run anywhere: Linux, Windows, Intel and Arm

Multi-architecture images target different operating systems or CPU architectures.  Docker will pull the right one for your machine when an image is requested.

## 16.1 - Why multi-architecture images are important
- Cloud workloads can be run much less expensively if using Arm
- [[Internet of Things|IoT]] usually run Arm processors

Examples below require setting `"experimental": true` in the Docker engine daemon config.

```powershell
# switch to the exercises folder:
cd ch16/exercises
# build for 64-bit Arm:
docker build -t diamol/ch16-whoami:linux-arm64 --platform linux/arm64 ./whoami
# check the architecture of the image:
docker image inspect diamol/ch16-whoami:linux-arm64 -f '{{.Os}}/{{.Architecture}}'
# and the native architecture of your engine:
docker info -f '{{.OSType}}/{{.Architecture}}'
```

<pre>
PS > docker image inspect diamol/ch16-whoami:linux-arm64 -f '{{.Os}}/{{.Architecture}}'
linux/arm64
PS > docker info -f '{{.OSType}}/{{.Architecture}}'
linux/x86_64
</pre>

The image is targeted for `linux\arm64`.  Since the .NET SDK image has an ARM variant, everything just works.

>[!NOTE]
> You can’t pull an image from a registry if there’s no variant to match your OS and CPU.

## 16.2 Building multi-arch images from one or more Dockerfiles

Two approaches:
	1.  Multi-stage, *single* Dockerfile that compiles the app from source.  If SDK and runtime images support all the architectures, you're good to go.
	2. Multiple Dockerfiles for the different architectures needing support.

These approaches aren't mutually exclusive.  You could have a single Dockerfile for `linux/amd64` and `linux/arm64` and then separate ones for `windows/amd64` and `linux/arm` (which is arm32).

Examples of using multiple files
```powershell
cd ./folder-list

# build for the native architecture - Intel/AMD:
docker image build -t diamol/ch16-folder-list:linux-amd64 -f ./Dockerfile.linux-amd64 .

# build for Arm 64-bit:
docker image build -t diamol/ch16-folder-list:linux-arm64 -f ./Dockerfile.linux-arm64 --platform linux/arm64 .

# and for Arm 32-bit
docker image build -t diamol/ch16-folder-list:linux-arm -f ./Dockerfile.linux-arm --platform linux/arm .

# run all the containers and verify the output:
docker container run diamol/ch16-folder-list:linux-amd64
docker container run diamol/ch16-folder-list:linux-arm64
docker container run diamol/ch16-folder-list:linux-arm
```

Multiple Dockerfiles are generally needed if you need to build a multi-arch version of a third-party app.  When using a single Dockerfile you have to be careful to use commands that work in both platforms.  Dockerfile may build correctly but will fail when run.
- `RUN` instructions fail at build time
- `CMD` aren't verified and will only fail when running.

Main architectures supported by Docker

| OS      | CPU       | Word Length | CPU Name | CPU Aliases           |
|---------|-----------|-------------|----------|-----------------------|
| Windows | Intel/AMD | 64-bit      | amd64    | x86_64                |
| Linux   | Intel/AMD | 64-bit      | amd64    | x86_64                |
| Linux   | Arm       | 64-bit      | arm64    | aarch64, armv8        |
| Linux   | Arm       | 32-bit      | arm      | arm32v7, armv7, armhf |


More information available in [Leverage multi-CPU architecture support](https://docs.docker.com/desktop/multi-arch/)

## 16.3 Pushing multi-arch images to registries with manifests

Manifests are required to have multi-arch images and have to be pushed to the registry.  They are pieces of metadata that links multiple image variants to the same tag.  They are created with the Docker CLI.

Will display manifest information for an image.
```powershell
docker manifest inspect diamol/base
```

Creates a manifest linking 3 architectures together
```powershell
# create a manifest with a name, followed by all the tags it lists:
docker manifest create "$dockerId/ch16-folder-list" "$dockerId/ch16-folder-list:linux-amd64" "$dockerId/ch16-folder-list:linux-arm64" "$dockerId/ch16-folder-list:linux-arm"

# push the manifest to Docker Hub:
docker manifest push "$dockerId/ch16-folder-list"

# now browse to your page on Docker Hub and check the image
```

## 16.4 - Building multi-arch images with Docker Buildx
There’s another way to run a Docker build farm that is more efficient and far easier to use, and that’s with a new feature of Docker called Buildx. Buildx is an extended version of the Docker build commands, and it uses a new build engine that is heavily optimized to improve build performance. It still uses Dockerfiles as the input and produces images as the output, so you can use it as a straight replacement for docker image build . Buildx really shines for cross-platform builds, though, because it integrates with Docker contexts, and it can distribute builds across multiple servers with a single command.

See the book for examples

Buildx makes these multi-arch builds very simple. You supply the nodes for each architecture you want to support, and Buildx can use them all, so your build commands don’t change whether you’re building for two architectures or 10. There’s an interesting difference with Buildx images on Docker Hub--there are no individual image tags for the variants, there’s just the single multi-arch tag. Compare that to the previous section where we manually pushed the variants and then added the manifest--all the variants had their own tags on Docker Hub, and as you build and deploy more image versions, that can get hard for users to navigate. If you don’t need to support Windows containers, Buildx is the best way to build multi-arch images right now.

## 16.5 - Understanding where multi-arch images fit in your roadmap

You may not need multi-arch images now, but you can evolve the process over time.  Future-proof yourself by always using multi-arch images in the `FROM` instructions and don't include any OS-specific commands in `RUN` or `CMD` instructions.  Complex deployment or startup logic can be built in a utility app using the same language as your application and be compiled in another stage of the build.