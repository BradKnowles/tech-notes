---
tags: [docker]
---

[[Docker in a Month of Lunches]]

## 2.1 - Running Hello World in a container

```powershell
docker container run diamol/ch02-hello-diamol
```

The docker container run command tells
Docker to run an application in a container.

<pre>
PS C:\Users\Brad> docker container run diamol/ch02-hello-diamol

<strong>Start a container from an application package (image) called diamol/ch02-hello-diamol.</strong>

Unable to find image 'diamol/ch02-hello-diamol:latest' locally
latest: Pulling from diamol/ch02-hello-diamol
31603596830f: Pull complete
93931504196e: Pull complete
d7b1f3678981: Pull complete
Digest: sha256:c4f45e04025d10d14d7a96df2242753b925e5c175c3bea9112f93bf9c55d4474
Status: Downloaded newer image for diamol/ch02-hello-diamol:latest

<strong>That image didn't exist on this machine, so Docker downloads it first.</strong>

---------------------
Hello from Chapter 2!
---------------------
My name is:
232702c13ff5
---------------------
Im running on:
Linux 5.10.60.1-microsoft-standard-WSL2 x86_64
---------------------
My address is:
inet addr:172.17.0.2 Bcast:172.17.255.255 Mask:255.255.0.0

<strong>Docker runs a container from the package and these logs are the output from the application.</strong>
</pre>

## 2.2 - What is a container?

A box with an app inside it.

It has its own **machine name** and **IP address**, and it also has its own **disk drive**
(Windows containers have their own Windows Registry too)

**The application inside the box can’t see anything outside the box**

Why is this so important? It fixes two conflicting problems in computing: **isolation** and
**density**. Density means running as many applications on your computers as possible,
to utilize all the processor and memory that you have.

Containers give you both. Each container shares the operating system of the computer running the container, and that makes them extremely lightweight. Containers
start quickly and run lean, so you can run many more containers than VMs on the same hardware

## 2.3 - Connecting to a container like a remote computer
```powershell
docker container run --interactive --tty diamol/base
```

`--interactive` - set up a connection to the container
`--tty` - connect to a terminal session inside the container

<pre>
PS C:\Users\Brad> docker container run --interactive --tty diamol/base
/ # hostname
c5dec3af0411
/ # date
Thu Mar  3 04:51:25 UTC 2022
/ #
</pre>

Get the details of all running containers
```powershell
docker container ls
```

<pre>
CONTAINER ID   IMAGE         COMMAND     CREATED        STATUS        PORTS     NAMES
f74565dc0944   diamol/base   "/bin/sh"   21 hours ago   Up 21 hours             dazzling_boyd
</pre>

> Container ID (which is assigned at random) is the same as the hostname inside the container.

List the processes running in a container
```powershell
docker container top <container id>
```

Display log entries the container has collected
```powershell
docker container logs <container id>
```

Show all the container details (inspect)
```powershell
docker container inspect <container id>
```

<pre>
[
    {
        "Id": "c5dec3af0411584e5dc542835cfff2be0fefc26b96f30ff417d93491ef500794",
        "Created": "2022-03-03T04:51:13.78561577Z",
        "Path": "/bin/sh",
        "Args": [],
        "State": {
            "Status": "exited",
            "Running": false,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 0,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2022-03-03T04:51:14.150599237Z",
            "FinishedAt": "2022-03-03T05:36:59.644788675Z"
        },
...
</pre>

## 2.4 - Hosting a website in a container

> Containers are running only while the application inside the container is running. As soon as the application process ends, the container goes into the exited state. Exited containers don’t use any CPU time or memory. 

> Containers don’t disappear when they exit. Containers in the exited state still exist, which means you can start them again, check the logs, and copy files to and from the container’s filesystem.

Run container that stays in background (detach)
```powershell
docker container run --detach --publish 8088:80 diamol/ch02-hello-diamol-web
```

--detach — Starts the container in the background and shows the container ID

--publish — Publishes a port from the container to the computer
Running a detached container just puts the container in the background

Publishing a container port means Docker listens for network traffic on the computer port, and then sends it into the container. In the preceding example, traffic sent to the computer on port 8088 will get sent into the container on port 80

[http://localhost:8088/]()

Show live view of how CPU, memory, network, and disk the container is using (stats)

```powershell
docker container stats
```

<pre>
CONTAINER ID   NAME              CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O   PIDS
e86081cde822   strange_satoshi   0.00%     4.613MiB / 31.22GiB   0.01%     2.48kB / 1.47kB   0B / 0B     82

</pre>

## 2.5 - Understanding how Docker runs containers

The [[Docker Engine]] is the management component of Docker.  It looks after the local image cache, downloading / reusing images.  It also works with the OS to create containers, virtual networks, and all other Docker resources.  It runs in the background.

The Docker Engine makes all the features available through the [[Docker API]], which is just a standard [[HTTP]]-based [[REST]] API.  You can configure the Engine to make the API accessible only from the local computer (which is the default), or make it available to other computers on your network.

The Docker command-line interface (CLI) is a client of the Docker API. When you run Docker commands, the CLI actually sends them to the Docker API, and the Docker Engine does the work.

> The only way to interact with the Docker Engine is through the API.

The Docker Engine uses a component called [[containerd]] to actually manage containers, and containerd in turn makes use of operating system features to create the virtual environment that is the container. 