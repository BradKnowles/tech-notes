---
tags: [docker/security,docker/remote,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Configuring Docker for secure remote access and CI/CD

The Docker CLI really only makes calls to the [[Docker API]].  It can be pointed to other remote destinations and not just the local machine.

> [!TIP]
> Remote access is how you administer test environments or debug issues in production.

## 15.1 - Endpoint options for the Docker API

The Docker CLI defaults to listening on a local channel, [[sockets]] on Linux and [[named pipes]] on Windows.  Both of these restrict traffic to the local machine.

Remote access to the [[Docker Engine]] can be enabled 3 ways
- Unsecured [[HTTP]] (**Don't ever do this**)
- [[Transport Layer Security|TLS]]
- [[Secure Shell|SSH]]

**Don't do this** except maybe for local testing.  **It gives anyone access to the machine to run whatever containers they want.**

Enabling the `Expose daemon on tcp://localhost:2375 without TLS` checkbox in Docker Desktop (or setting the hosts property in the `daemon.json` file) will turn on remote access locally.

Connecting to a Docker instance on another host
```powershell
# connect to the local Engine over TCP:
docker --host tcp://localhost:2375 container ls
```

> [!DANGER]
> Don’t underestimate how dangerous this is. Linux containers use the same user accounts as the host server, so if you run a container as the Linux admin account, `root`, you’ve pretty much got admin access to the server. Windows containers work slightly differently, so you don’t get unlimited server access from within a container, but you can still do unpleasant things.

## 15.2 - Configuring Docker for secure remote access

The two secure and recommended ways to get access to a remote docker engine is via [[Transport Layer Security|TLS]] and [[Secure Shell|SSH]].

> [!NOTE]
> Securing access to Docker Engine requires  access to the machine running Docker.

### Transport Layer Security (TLS)
- Most widely used
- Requires management overhead in rotating certificates for the server and client
	- Certificates are created with a lifespan, so they can be short-lived to restrict access
- Requires certificate and key file pair
	- One for the Docker API
	- One for the client
	- Key file acts as password for the certificate
- Requires configuration changes to Docker engine


### Secure Shell (SSH)
- Requires SSH client
- Easier to manage who has machine access
- No certificates required
- Doesn't require configuration changes to Docker engine
- **Using this option gives the user access to the whole server**, which most operations people will not be a fan.

### Summary
- TLS might be a better option in spite of the certificate overhead because it’s all handled within Docker and it doesn’t need an SSH server or client.
- There's no authorization or auditing with these methods.  There's no way to restrict what they can do and no way to know who did what.
- Remember which environment the Docker CLI is using can be difficult and commands may inadvertently be run against the wrong environment.  This is made easier with `contexts`.  It stores connection details for an engine.

>[!INFO]
>Securing your Engine is about two things: encrypting the traffic between the CLI and API, and authenticating to ensure the user is allowed access to the API. There’s no authorization—the access model is all or nothing. If you can’t connect to the API, you can’t do anything, and if you can connect to the API, you can do everything.