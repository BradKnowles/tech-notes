---
tags: [docker/configuration,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Application configuration management  in containers

Two main way configuration is handled in in containers, [[environment variables]] and files on disk.

## 18.1 - A multi-tiered approach to app configuration

Configuration is usually of three types
- Release-level
	- The same for every environment in a given release
- Environment-level
	- Different per environment
- Feature-level
	- Change behavior between releases

Examples from a Node.js app
```powershell
cd ch18/exercises/access-log

# run a container with the default config in the image:
docker container run -d -p 8080:80 diamol/ch18-access-log

# run a container loading a local config file override:
docker container run -d -p 8081:80 -v "$(pwd)/config/dev:/app/config-override" diamol/ch18-access-log

# check the config APIs in each container:
curl http://localhost:8080/config
curl http://localhost:8081/config
```
<pre>
PS ch18\exercises\access-log> curl http://localhost:8080/config

{"release":"19.12","environment":"UNKNOWN","metricsEnabled":true}

PS ch18\exercises\access-log> curl http://localhost:8081/config

{"release":"19.12","environment":"DEV","metricsEnabled":false}
PS ch18\exercises\access-log>
</pre>

The first container used the default config file from the image.  The second used the override from the file system bind mount.  It could have easily been a config object or secret from the container cluster.

This  merges from the default file in the image, the local config override file in the volume, and the specific environment variable setting.
```powershell
cd ch18/exercises/access-log

# run a container with an override file and an environment variable:
docker container run -d -p 8082:80 -v "$(pwd)/config/dev:/app/config-override" -e NODE_CONFIG='{\"metrics\": {\"enabled\":\"true\"}}' diamol/ch18-access-log

# check the config:
curl http://localhost:8082/config
```

## 18.2 - Packaging config for every environment

Some systems package all the config entries for all the environments in a single image.  Setting the environment name allows for the proper base file and override to be applied.  .NET appsettings work this way.
- `appsettings.json` - the base for all environments
- `appsettings.{Environment}.json` - the override for for the named environment
- Environment variables - specifies the environment and any other overrides.

## 18.3 - Loading configuration from the runtime

