---
tags: [docker/volumes,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Using Docker volumes for persistent storage

## 6.1 - Why data in containers is not permanent

Docker containers have filesystems with a single drive. Images are stored as layers so the container filesystem is actually a virtual filesystem that merge those layers together.

Each container's filesystem of other containers, even from the same image.

Run two container from the same image
```powershell
docker container run --name rn1 diamol/ch06-random-number
docker container run --name rn2 diamol/ch06-random-number

# Copy the text file from both containers
docker container cp rn1:/random/number.txt number1.txt
docker container cp rn2:/random/number.txt number2.txt
 
cat number1.txt
cat number2.txt
```

Both containers wrote a random number to `/random/number.txt` with different contents.

On Windows the filesystem is `C:\` on Linux it's `\dev\sda1`.

The image layers are shared between containers so they are read-only, but each container has one writable layer that has the same lifecycle as the container.  **The container's writeable filesystem layer stays around until the container is removed.**  Which means even if the container is stopped, the filesystem is still around and can be accessed.

If a container edits a file from one of the read-only image layers, it is copied to the writeable layer and edited there.

Start a container, create a local file, copy the local file into container, see output change
```powershell
docker container run --name f1 diamol/ch06-file-display

echo "http://eltonstoneman.com" > url.txt

docker container cp url.txt f1:/input.txt

docker container start --attach f1
```

Create a 2nd container, see the file output is back to the original. Remove container `f1`, try to copy file from `f1`, see the data is gone.
```powershell
docker container run --name f2 diamol/ch06-file-display
docker container rm -f f1
docker container cp f1:/input.txt .
```

> The container filesystem has the same life cycle as the container, so when the container is removed, the writeable layer is removed, and **any changed data in the container is lost**.

## 6.2 - Running containers with Docker volumes

Volumes are removable storage for containers, like a USB drive.  They exist outside of a container and have their own lifecycle.  They mount to the container as a directory.  When the container stops and is removed, a new container can use the same volume and the data is still there.

Volumes can be manually created or can use the `VOLUME` instruction in a Dockerfile.

Example Dockerfile
```dockerfile
FROM diamol/dotnet-aspnet
WORKDIR /app

ENTRYPOINT ["dotnet", "ToDoList.dll"]
VOLUME /data
COPY --from=builder /out/ .
```

The container will have a directory at `/data` in Linux, `C:\data` in a Windows container.

Run a container, see the volume that was created
```powershell
docker container run --name todo1 -d -p 8010:80 diamol/ch06-todo-list
docker container inspect --format '{{.Mounts}}' todo1
docker volume ls
 ```
 <pre>
 [{volume 3925bcd327b7060bc94ddef3a8b24c56c23aed36a55275a37ea1af24f1e5ecba /var/lib/docker/volumes/3925bcd327b7060bc94ddef3a8b24c56c23aed36a55275a37ea1af24f1e5ecba/_data /data local  true }]
 </pre>

 Run the to-do app - [http://localhost:8010](http://localhost:8010)

 If declared in a Docker image, a new volume will be created for each new container.

 Start a 2nd container see data is not shared
 ```powershell
# this new container will have its own volume
docker container run --name todo2 -d diamol/ch06-todo-list

# on Linux:
docker container exec todo2 ls /data

# on Windows:
docker container exec todo2 cmd /C "dir C:\data"

# this container will share the volume from todo1
docker container run -d --name t3 --volumes-from todo1 diamol/ch06-todo-list

# on Linux:
docker container exec t3 ls /data

# on Windows:
docker container exec t3 cmd /C "dir C:\data"
```

The `todo2` container starts with a new volume, so the `/data` is empty. While `t3` shows the same `todo-list.db` from `todo1`.

> Sharing volumes between containers is straightforward, but it's probably **not what you want to do**. Apps that write data typically expect exclusive access to the files, and they may not work correctly (or at all) if another container is reading and writing to the same file at the same time. Volumes are better used to preserve state between application upgrades, and then it's **better to explicitly manage the volumes**. You can create a named volume and attach that to the different versions of your application container.

Create a named volume, share it between containers
```powershell
# save the target file path in a variable:
$target='/data' # for Linux containers
# $target='c:\data' # for Windows containers

# create a volume to store the data:
docker volume create todo-list

# run the v1 app, using the volume for app storage:
docker container run -d -p 8011:80 -v todo-list:$target --name todo-v1 diamol/ch06-todo-list

# add some data through the web app at http://localhost:8011

# remove the v1 app container:
docker container rm -f todo-v1

# and run a v2 container using the same volume for storage:
docker container run -d -p 8011:80 -v todo-list:$target --name todo-v2 diamol/ch06-todo-list:v2
```

> The `VOLUME` instruction in the Dockerfile and the `volume` (or `v`) flag for running containers are separate features. Images built with a `VOLUME` instruction will **always create a volume** for a container if there is **no `volume` (`v`) specified in the run command**. The volume will have a random ID, so you can use it after the container is gone, but only if you can work out which volume has your data.

> The `volume` flag mounts a volume into a container whether the **image has a volume specified or not**. If the image does have a volume, the volume flag can override it for the container by using an existing volume for the same **target path**--so a new volume won't be created.

> As an image author, you should use the `VOLUME` instruction as a fail-safe option for stateful applications. That way containers will always write data to a persistent volume even if the user doesn't specify the `volume` flag.

> As an image user, it's better not to rely on the defaults and to work with named volumes.

## 6.3 - Running containers with filesystem mounts

A bind mount is a directory on the host shared with a container. It is bidirectional.

Create a bind mount, use `curl` to force app to create database, see `todo-list-db` in host filesystem
```powershell
$source="$(pwd)\databases".ToLower(); $target="/data"

mkdir ./databases

docker container run --mount type=bind,source=$source,target=$target -d -p 8012:80 diamol/ch06-todo-list

curl http://localhost:8012

ls ./databases
```

There are some security concerns here.  In order to write files on the host, the container needs elevated permissions.  bind mounts can be read-only and is good for sharing configuration settings from the host into the application container.

Override the container configuration with a read-only bind mount.
```powershell
cd ./ch06/exercises/todo-list
 
# save the source path as a variable:
$source="$(pwd)\config".ToLower(); $target="/app/config"

# run the container using the mount:
docker container run --name todo-configured -d -p 8013:80 --mount type=bind,source=$source,target=$target,readonly diamol/ch06-todo-list

# check the application:
curl http://localhost:8013

# and the container logs:
docker container logs todo-configured

```

Bind mounts can be from any source the host computer has access to, shared network drives, etc.

## 6.4 - Limitations of filesystem mounts

### Mounting a target *directory* that already exists and has files from image layers
The source of the bind mount overrides the image target.  **The original files from the image are no longer available**.

### Mounting a *single file* to a target *directory* that already exists.
The directory contents are merged.  The original image files AND the new file from the host are available, except in Windows containers.

### Mounting a *distributed filesystem*
This scenario is less common.  Think Azure Blob Storage or AWS S3 or SMB file shares on a local network.  You can mount these into a container and the mount will look normal.  **If the distributed filesystem doesn't support the same operations, the app could fail.**

## 6.5 - Understanding how the container filesystem is built
- Every container has a single disk, which Docker manages via the union filesystem.
- The union filesystem lets the container see a single drive, no matter where the file is sourced from.

### Guidelines for using the storage options.
- Writeable layer - Perfect for short-term storage, like caching data to disk to save on network calls or computations. These are unique to each container but are gone forever when the container is removed.
- Local bind mounts - Used to share data between the host and the container. Developers can use bind mounts to load the source code on their computer into the container, so when they make local edits to HTML or JavaScript files, the changes are immediately in the container without having to build a new image.
- Distributed bind mounts - Used to share data between network storage and containers. These are useful, but you need to be aware that network storage will not have the same performance as local disk and may not offer full filesystem features. They can be used as read-only sources for configuration data or a shared cache, or as read-write to store data that can be used by any container on any machine on the same network.
- Volume mounts - Used to share data between the container and a storage object that is managed by Docker. These are useful for persistent storage, where the application writes data to the volume. When you upgrade your app with a new container, it will retain the data written to the volume by the previous version.
- Image layers - These present the initial filesystem for the container. Layers are stacked, with the latest layer overriding earlier layers, so a file written in a layer at the beginning of the Dockerfile can be overridden by a subsequent layer that writes to the same path. Layers are read-only, and they can be shared between containers.
