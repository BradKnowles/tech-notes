---
tags: [docker/registries,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Sharing images with Docker Hub and other registries

[[Docker Hub]] is the default Docker registry.  A registry is the server that stores images centrally.

## 5.1 - Working with registries, repositories, and image tags

<pre>
<span style="color : Green">docker.io</span>/<span style="color : Gold">diamol</span>/<span style="color : Cornsilk">golang</span>:<span style="color : DarkSalmon">latest</span>
</pre>

Four parts to a Docker image name
- Domain of the registry - `docker.io`
	- Default is Docker Hub, docker.io
- Account of image owner - `diamol`
- Image repo name - `golang`
	- Used for application name
	- One repo can store many versions of an image
- Image tag - `latest`
	- Used for versioning or variations of an application. `Latest` is the default.

Locally, naming isn't as important, but it's very important when sharing with a registry.

> When you start building your own application images, you should always tag them. Tags are used to identify different versions of the same application.

## 5.2 - Pushing your own images to Docker Hub

> CLI login is done via Access Tokens. Need to create on the website before pushing to registry.

```powershell
docker login --username $dockerId
```
<pre>
Password:
Login Succeeded
</pre>

Create a new tag for an existing image
```powershell
docker image tag image-gallery $dockerId/image-gallery:v1
```

List image references
```powershell
docker image ls --filter reference=image-gallery --filter reference='*/image-gallery'
```
<pre>
REPOSITORY                TAG       IMAGE ID       CREATED        SIZE
$dockerId/image-gallery   <em>v1</em>        <strong>4d042ef0c7d0</strong>   12 hours ago   27.1MB
image-gallery             <em>latest</em>    <strong>4d042ef0c7d0</strong>   12 hours ago   27.1MB
</pre>

Same image IDs but with a tag of `v1` and `latest`.

Push image to registry
```powershell
docker image push $dockerId/image-gallery:v1
```
<pre>
The push refers to repository [docker.io/$dockerId/image-gallery]
85c9a9923d80: Pushed
0113d4011c4b: Pushed
6200b3092267: Pushed
7f0fb96d9c37: Pushed
f87269b94a71: Mounted from diamol/ch03-lab
89ae5c4ee501: Mounted from diamol/ch03-lab
v1: digest: sha256:ce1a203c1b48d46f32254f4a9573e15f3369d837975bbed3c30d00f899a880fc size: 1574
</pre>

Docker registries also work on the idea of image layers. The image is pushed, but the layers are uploaded.

> Layers are only uploaded if there isn't an existing layer with the same hash. This applies to **ALL** the image layers across the whole registry

If you optimize to the point where 90% of layers come from the cache when you build, 90% of those layers will already be in the registry when you push. Optimized Dockerfiles reduce build time, disk space, and network bandwidth.

Docker Hub creates a new repo if necessary and by default that repository has **public read rights**. Anyone can find and use this image.

## 5.3 - Running and using your own Docker registry

Running your own registry on your own local network can cut down on bandwidth and transfer times.  It also allows you to keep control over the data.  It can also serve as a backup if the main registry goes offline.

Docker provide the Dockerfile for main registry server on [GitHub](https://github.com/docker/distribution-library-image).  The source code is in the [/distribution/distribution](https://github.com/distribution/distribution) repo.

```powershell
# run the registry with a restart flag so the container gets
# restarted whenever you restart Docker:
docker container run -d -p 5000:5000 --restart always diamol/registry
```

Tag image for the local registry
```powershell
docker image tag image-gallery localhost:5000/gallery/ui:v1
```

To get around the lack of [[HTTPS]] on the local registry, add `"insecure-registries": ["localhost:5000"]` to the end of the `daemon.json` file in 'C:\\Users\\&ltuser&gt\\.docker`. Restart Docker desktop.

Push image to local registry
```powershell
docker image push localhost:5000/gallery/ui:v1
```
<pre>
85c9a9923d80: Pushed
0113d4011c4b: Pushed
6200b3092267: Pushed
7f0fb96d9c37: Pushed
f87269b94a71: Pushed
89ae5c4ee501: Pushed
v1: digest: sha256:ce1a203c1b48d46f32254f4a9573e15f3369d837975bbed3c30d00f899a880fc size: 1574
PS
</pre>

## 5.4 - Using image tags effectively
Most projects use a form of [[Semantic Versioning]] to communicate how much the software has changed from version to version.  Using a similar structure to images allows users to be in control of their dependencies.

Tagging with multiple versions structures
```powershell
docker image tag image-gallery localhost:5000/gallery/ui:latest
docker image tag image-gallery localhost:5000/gallery/ui:2
docker image tag image-gallery localhost:5000/gallery/ui:2.1
docker image tag image-gallery localhost:5000/gallery/ui:2.1.106
```

When the application move to version 2.2.31 the tags look like this.
```powershell
docker image tag image-gallery localhost:5000/gallery/ui:latest
docker image tag image-gallery localhost:5000/gallery/ui:2
docker image tag image-gallery localhost:5000/gallery/ui:2.2
docker image tag image-gallery localhost:5000/gallery/ui:2.2.31
```

Dockerfiles that are using the `2` tag always get the latest `2.x` version.  Dockerfiles using the `2.2` tag get all the `2.2.x` patch updates but not minor updates.

> It's especially important to use **specific image tags** for the base images in your own Dockerfiles. It's great to use the product team's build tools image to build your apps and their runtime image to package your apps, but if you don't specify versions in the tags, you're setting yourself up for trouble in the future. A new release of the build image could break your Docker build. Or worse, a new release of the runtime could break your application.

## 5.5 - Turning official images into golden images

Anyone can push to Docker Hub.  How can you trust those images?

Verified Publishers and Official Images

### Verified Publishers
Companies like Microsoft, Oracle, and IBM go through an more stringent approval process and security review.  Favor these if you can.

### Official Images
Usually for [[Open Source]] projects and are jointed maintained by the project team and Docker.  The images are scanned for vulnerabilities and the Dockerfiles are on GitHub.

### Golden Images
These are images that use images from verified publishers or official images as their base and add whatever custom setup is necessary.  These live in your companies registry.

There's nothing special about golden images. They start with a Dockerfile, and that builds an image with your own reference and naming scheme.

The official images may have a new release every month, but you can choose to restrict your golden images to quarterly updates. And golden images open up one other possibility--you can enforce their use with tools in your continuous integration (CI) pipeline: Dockerfiles can be scanned, and if someone tries to build an app without using golden images, that build fails. It’s a good way of locking down the source images teams can use.
