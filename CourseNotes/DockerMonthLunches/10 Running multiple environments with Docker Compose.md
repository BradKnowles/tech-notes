---
tags: [docker,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Running multiple environments with Docker Compose

Portability is one of Docker's major benefits.  Applications in containers work the same way, no matter where they are deployed.  That keeps the application in sync between environments, also known as eliminating drift.

Drift is common in manually deployed applications; updates don't match in every environments, dependencies are slightly off, etc.

Docker solves that by packaging up all the dependencies and proper tool versions in one container.  That container becomes the deployable artifact.

However, sometimes applications do need to act differently in different environments and [[Docker Compose]] provides advanced features for that purpose.

## 10.1 - Deploying many applications with Docker Compose

> Docker Compose is a tool for running multi-container applications on a single Docker Engine. It's great for developers, and it's also heavily used for non-production environments. Organizations often run multiple versions of an app in different environments--maybe version 1.5 is running in production, version 1.5.1 is being tested in a hotfix environment, version 1.6 is finishing up user testing, and version 1.7 is in system test. Those non-production environments don't need the scale and performance of production, so it's a great use case for Docker Compose to run those environments and get maximum utilization from your hardware.

Running those on the same hardware requires adjustments.  For example, they all can't be listening on port 80 or writing files to same locations.

To run several copies of the same application you have to work around the default Docker resource setup of naming conventions and labels.

Example of trying to run multiple copies of the same application
```powershell
cd ./ch10/exercises
 
# run the random number app from chapter 8:
docker-compose -f ./numbers/docker-compose.yml up -d

# run the to-do list app from chapter 6:
docker-compose -f ./todo-list/docker-compose.yml up -d

# and try another copy of the to-do list:
docker-compose -f ./todo-list/docker-compose.yml up -d
```

Docker sees `todo-list` as already running and does not start any new containers.

Project names are who Docker Compose keeps track of which resources are part of the same application.  By default, the project name is the name of the directory that contains the Compose file.  Compose prefix the project name when it creates resources and adds a numeric counter to containers as a suffix.

<pre>
<span style="color : Green">todo-list</span>-<span style="color : Gold">todo-web</span>-<span style="color : Blue">1</span>
</pre>

|Part       | Meaning                                          |
|-----------|--------------------------------------------------|
| todo-list | Name of directory                                |
| todo-web  | Service name in Compose file, also used for DNS  |
| 1         | Container index. Incremented as containers scale |

By add `-p` to `docker compose up` you can specify a different project name and have multiple copies of the application running

```powershell
# and try another copy of the to-do list:
docker-compose -f ./todo-list/docker-compose.yml -p todo-test up -d
```

## 10.2 - Using Docker Compose override files

Teams often try to use multiple Compose files to configure multiple environments.  This can lead to the same drift from traditional deployments.

Docker Compose will merge multiple Compose files together.  This allows you to start with a core `docker-compose.yml`  and use `docker-compose-<env>.yaml` files to merge environment specific settings into the final output.

```yaml
# from docker-compose.yml - the core app specification:
version: "3.7"

services:
  todo-web:
    image: diamol/ch06-todo-list
    ports:
      - 80
    environment:
      - Database:Provider=Sqlite
    networks:
      - app-net

networks:
  app-net:

# and from docker-compose-v2.yml - the version override file:
version: "3.7"

services:
  todo-web:
    image: diamol/ch06-todo-list:v2
```

Since `docker-compose-v2.yml` overrides the image tag, when the `up` command is executed, and both files are specified using `-f`, the `diamol/ch06-todo-list:v2` image will be used.

To inspect the output of the merged files, use the `config` command.  It validates the configuration.

```powershell
docker-compose -f ./todo-list/docker-compose.yml -f ./todo-list/docker-compose-v2.yml config
```

Run multiple copies of an application
```powershell
# remove any existing containers
docker container rm -f $(docker container ls -aq)

# run the app in dev configuration:
docker-compose -f ./numbers/docker-compose.yml -f ./numbers/docker-compose-dev.yml -p numbers-dev up -d

# and the test setup:
docker-compose -f ./numbers/docker-compose.yml -f ./numbers/docker-compose-test.yml -p numbers-test up -d

# and UAT:
docker-compose -f ./numbers/docker-compose.yml -f ./numbers/docker-compose-uat.yml -p numbers-uat up -d
```

This creates three applications, `numbers-dev`, `numbers-test`, and `numbers-uat`.

If you run `docker network ls` you'll see three separate networks have also been created.  This is because each `docker-compose-<env>.yml` files overrides the base network name.

<pre>
NETWORK ID     NAME           DRIVER    SCOPE
2485978db83a   bridge         bridge    local
025a9c05bec3   host           host      local
10dba6a95983   none           null      local
<strong>24cfac357b62   numbers-dev    bridge    local
63ce3e179e34   numbers-test   bridge    local
2c560a750f88   numbers-uat    bridge    local</strong>
</pre>

There are three different URLs as well, 

| Environment | URL                                              |
|-------------|--------------------------------------------------|
| uat         | [http://localhost/](http://localhost/)           |
| test        | [http://localhost:8080/](http://localhost:8080/) |
| dev         | [http://localhost:8088/](http://localhost:8088/) |

Even though the services have the same names, they are on separate networks, so the `uat` services can't talk to `test` etc.

> Reminder that Docker Compose is a client-side tool, and you need access to all your Compose files to manage your apps. You also need to remember the project names you used. If you want to clear down the test environment, removing containers and the network, you would normally just run `docker-compose down`, but that won't work for these environments because Compose needs all the same file and project information you used in the `up` command to match the resources.

## 10.3 - Injecting configuration with environment variables and secrets
Compose files have a `secrets` section and `services` has a `secrets` property.

### Secrets

```yaml
version: "3.7"

services:
  todo-web:
    image: diamol/ch06-todo-list
    secrets:
      - source: todo-db-connection
        target: /app/config/secrets.json

secrets:
 todo-db-connection:
 file: ./config/secrets.json
```

Inside the container, the secrets will be loaded from `/app/config/secrets.json`, the `secrets` section specifies where on the host system to read those secrets `./config/secrets.json`.

Secrets are useful as they are supported in Docker Compose, [[Docker Swarm]] and [[Kubernetes]].

The secret specifies the `todo-db-connection` value, which mean that secret needs to be configured in the Compose file.  The `secrets` section tells the container runtime to get the value of `todo-db-connection` from `./config/secrets.json`.

> The key thing to remember here, is all this does is add files to the container.  Your app has to know about the files and know how to read from them.  In the Todo app, the `Startup.cs` is configured to pull from `secrets.json`

```csharp
config.AddJsonFile("config/config.json", optional: true)
	  .AddJsonFile("config/secrets.json", optional: true);
```

### Environment Variables

```yaml
version: "3.7"

services:
  todo-web:
    ports:
      - 8089:80
    environment:
      - Database:Provider=Sqlite
    env_file:
      - ./config/logging.debug.env

secrets:
  todo-db-connection:
    file: ./config/empty.json
```

- `environment` adds an environment variable inside the container. This setting configures the app to use the SQLite database, which is a simple data file. This is the easiest way to set configuration values, and it's clear from the Compose file what's being configured.
- `env_file` contains the path to a text file, and the contents of the text file will be loaded into the container as environment variables. Each line in the text file is read as an environment variable, with the name and value separated by an equals sign. The contents of this file set the logging configuration. Using an environment variable file is a simple way to share settings between multiple components, because each component references the file rather than duplicates a list of environment variables.

```ini
# logging.debug.env

Logging__LogLevel__Default=Debug
Logging__LogLevel__System=Debug
Logging__LogLevel__Microsoft=Debug
```

If Compose finds a file called `.env` in the current folder, it will treat that as an environment file and read the contents as a set of environment variables, populating them before it runs the command.

```ini
#.env file in directory

# container configuration - ports to publish:
TODO_WEB_PORT=8877
TODO_DB_PORT=5432

# compose configuration - files and project name:
COMPOSE_PATH_SEPARATOR=;
COMPOSE_FILE=docker-compose.yml;docker-compose-test.yml
COMPOSE_PROJECT_NAME=todo_ch10
```

When `docker compose up` is executed, it will read from the `.env` files and use the values from `COMPOSE_FILE` instead of having to specific `-f docker-compose.yml -f docker-compose-test-yml`.  It will also use `COMPOSE_PROJECT_NAME` to name the project instead of using `-p todo_ch10`.

Partial `docker-compose-test.yml`
```yaml
version: "3.7"

services:
  todo-web:
    ports:
      - "${TODO_WEB_PORT}:80"
    environment:
      - Database:Provider=Postgres
    env_file:
      - ./config/logging.information.env
    networks:
      - app-net

  todo-db:
    image: diamol/postgres:11.5
    ports:
      - "${TODO_DB_PORT}:5432"
    networks:
      - app-net
```

Additionally, `${TODO_WEB_PORT}` will be replaced by `8877` and `${TODO_DB_PORT}` will be replaced by `5432` without having to be specified at the command line.

> Be aware that Docker Compose only looks for a file called `.env`. You can't specify a filename, so you can't easily switch between environments with multiple environment files.

### Summary
- Using the `environment` property to specify environment variables is the simplest option, and it makes your application configuration easy to read from the Compose file. Those settings are in plain text, though, so you shouldn't use them for sensitive data like connection strings or API keys.
- Loading configuration files with `secret` properties is the most flexible option, because it's supported by all the container runtimes and it can be used for sensitive data. The source of the secret could be a local file when you're using Compose, or it could be an encrypted secret stored in a Docker Swarm or Kubernetes cluster. Whatever the source, the contents of the secret get loaded into a file in the container for the application to read.
- Storing settings in a file and loading them into containers with the `environment_file` property is useful when you have lots of shared settings between services. Compose reads the file locally and sets the individual values as environment properties, so you can use local environment files when you're connected to a remote Docker Engine.
- The Compose environment file, `.env`, is useful for capturing the setup for whichever environment you want to be the default deployment target.

## 10.4 - Reducing duplication with extension fields

Using extension fields to define blocks of YAML in a single place, which you can reuse throughout the Compose file. Extension fields are a powerful but underused feature of Compose. They remove a lot of duplication and potential for errors, and they're straightforward to use once you get accustomed to the YAML merge syntax.

Defining extension fields
```yaml
version: "3.7"

x-logging: &logging
  logging:  
    options:
      max-size: '100m'
      max-file: '10'

x-labels: &labels
  app-name: image-gallery
```

Using the conventional prefix `x-` you define blocks that can be reused later.

```yaml
services:
  accesslog:
    <<: *logging
    labels:
      <<: *labels

  iotd:
    ports:
      - 8080:80
    <<: *logging
    labels:
      <<: *labels
      public: api
```

Using [[YAML]] merge syntax `<<:` followed by the named defined in the `&` field from above, Compose will replace `<<: *logging` with
```yaml
  logging:  
    options:
      max-size: '100m'
      max-file: '10'
```

Use the `config` command to see this in action.

Resulting YAML output
```yaml
services:
  accesslog:
    image: diamol/ch09-access-log
    labels:
      app-name: image-gallery
    logging:
      options:
        max-file: "10"
        max-size: 100m
    networks:
      app-net: null
```

`accesslog` now has a fully populated `logging` section, matching the `&logging` extension field.  The `labels` section has the entry from the `&labels` field.

> There is one big limitation, though: extension fields don’t apply across multiple Compose files, so you can’t define an extension in a core Compose file and then use it in an override. That’s a restriction of YAML rather than Compose

## 10.5 - Understanding the configuration workflow with Docker

Having the entire deployment configuration living in source control allows for deployment of any version and running the right set of deployment scripts.  It also allows developers to run the production stack locally to quickly fix bugs.

Docker Compose has mechanism to capture the differences in environments, but yet still use the same container image, which is critically important.

Docker Compose helps in three key areas:
- Application composition -- Using override files developers can run the pieces of the stack that are necessary for their work.
- Container configuration -- Overrides also allow for tweaks to container configuration, published ports, volume paths, local drives, etc.  This helps running multiple environments on the same machine.
- Application Configuration -- It's quite common for the app to behave differently in various environments.  Overrides, environment files, and/or secrets can be used to change logging levels, change the size of caches, point to different databases, etc.

> The most important takeaway from this is that the configuration workflow uses the same Docker image in every environment. The build process will produce a tagged version of your container images, which have passed all the automated tests.
