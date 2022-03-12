---
tags: [docker,diamol,100DaysOfCode]
---

source: [[Docker in a Month of Lunches]]

# Adding observability with containerized monitoring

> Observability is a critical piece of the software landscape when you're running applications in containers--it tells you what your applications are doing and how well they're performing, and it can help you pinpoint the source of problems.


## 9.1 - The monitoring stack for containerized applications

The dynamic nature of containers means monitoring them is different than a traditional environment of servers.  Monitoring tools have to be container aware and be able to discover containers using the container platform.

The tools for this are [[Prometheus]] and [[Grafana]].

> Prometheus brings one very important aspect to monitoring: consistency. You can export the same type of metrics for all your applications, so you have a standard way to monitor them whether they're .NET apps in Windows containers or Node.js apps in Linux containers. You only have one query language to learn, and you can apply it for your whole application stack.

Docker itself can export metrics to Prometheus.  Add these settings to the JSON properties in the *Docker Engine* section of Docker Desktop's setting.

```json
"metrics-addr" : "0.0.0.0:9323",
"experimental": true
```

Metrics will be published on port `9323`.  Browse to [http://localhost:9323/metrics](http://localhost:9323/metrics) to see the Docker metrics.

<pre>
<strong># HELP builder_builds_failed_total Number of failed image builds</strong>
# TYPE builder_builds_failed_total counter
builder_builds_failed_total{reason="build_canceled"} 0
builder_builds_failed_total{reason="build_target_not_reachable_error"} 0
builder_builds_failed_total{reason="command_not_supported_error"} 0
builder_builds_failed_total{reason="dockerfile_empty_error"} 0
builder_builds_failed_total{reason="dockerfile_syntax_error"} 0
builder_builds_failed_total{reason="error_processing_commands_error"} 0
builder_builds_failed_total{reason="missing_onbuild_arguments_error"} 0
builder_builds_failed_total{reason="unknown_instruction_error"} 0
<strong># HELP builder_builds_triggered_total Number of triggered image builds</strong>
# TYPE builder_builds_triggered_total counter
builder_builds_triggered_total 0
<strong># HELP engine_daemon_container_actions_seconds The number of seconds it takes to process each container action</strong>
# TYPE engine_daemon_container_actions_seconds histogram
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.005"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.01"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.025"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.05"} 1
engine_daemon_co
</pre>

The output is Prometheus's own format.  It's simple text-based and has some help text for each metric.

Prometheus runs scheduled jobs to pull metrics and stores them in it's own database with a timestamp.  There's a basic UI as well for testing queries.

## 9.2 Exposing metrics from your application

Instrumenting your own apps, takes some additional work, but not a lot and the visibility it provides is extremely worth it.

You can export any metric you need to track, but you must write the code yourself.

An example using Node.js
```js
//declare custom metrics:
const accessCounter = new prom.Counter({
    name: "access_log_total",
    help: "Access Log - total log requests"
});
 
const clientIpGauge = new prom.Gauge({
    name: "access_client_ip_current",
    help: "Access Log - current unique IP addresses"
});
 
//and later, update the metrics values:
accessCounter.inc();
clientIpGauge.set(countOfIpAddresses);
```

>Each Prometheus client library works in a different way.

There are [Counter, Gauge, Histogram and Summary metrics](https://prometheus.io/docs/concepts/metric_types/) and it's up to you to choose which one are appropriate for your application.

### Guidelines on what to capture
- When you talk to external systems, record how long the call took and whether the response was successful--you'll quickly be able to see if another system is slowing yours down or breaking it.
- Anything worth logging is potentially worth recording in a metric--it's probably cheaper on memory, disk, and CPU to increment a counter than to write a log entry, and it's easier to visualize how often things are happening.
- Any details about application or user behaviors that business teams want to report on should be recorded as metrics--that way you can build real-time dashboards instead of sending historical reports.

## 9.3 - Running a Prometheus container to collect metrics

Prometheus pulls data from your app to collect metrics.  It's called scraping.  You have to configure the endpoints you want Prometheus to scrape.

In production, Prometheus can be configured to find all the containers.  With Docker Compose on a single server, using the service names will work with Docker's [[Domain Name System|DNS]].

Example configuration
```yaml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: "image-gallery"
    metrics_path: /metrics
    static_configs:
      - targets: ["image-gallery"]

  - job_name: "iotd-api"
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["iotd"]

  - job_name: "access-log"
    metrics_path: /metrics
    scrape_interval: 3s
    dns_sd_configs:
      - names:
          - accesslog
        type: A
        port: 80
        
  - job_name: "docker"
    metrics_path: /metrics
    static_configs:
      - targets: ["DOCKER_HOST:9323"]
```

This configuration pols every container every 10 seconds, uses DNS for the container IP addresses.

> For the `image-gallery` it only expects to find a single container, so you'll get unexpected behavior if you scale that component.

Prometheus always uses the first IP address if DNS returns multiple values.  This will end up reporting metrics from different containers as Docker load balances the endpoints.

`acesslog` is configured to support multiple IP addresses.  Prometheus will use all the IP addresses and poll them on the same schedule.

```powershell
cd ..\ch09\exercises\

# loop to make 10 HTTP GET request - on Windows:
for ($i=1; $i -le 10; $i++) { iwr -useb http://localhost:8010 | Out-Null }
      
# or on Linux:
for i in {1..10}; do curl http://localhost:8010 > /dev/null; done

```

Browse to [http://localhost:9090/graph](http://localhost:9090/graph) and select `access_log_total` and click `Execute`.  This will show the totals broken down by container instance.

## 9.4 - Running a Grafana container to visualize metrics

For more powerful visualizations, you'll want to use [[Grafana]].

```powershell
# load your machine's IP address into an environment variable - on Windows:
$env:HOST_IP=$(Get-NetIPConfiguration | Where-Object {$_.IPv4DefaultGateway -ne $null -and $_.NetAdapter.Status -ne "Disconnected"}).IPv4Address.IPAddress

# run the app with a Compose file which includes Grafana:
docker-compose -f ./docker-compose-with-grafana.yml up -d --scale accesslog=3

# now send in some load to prime the metrics - on Windows:
for ($i=1; $i -le 20; $i++) { iwr -useb http://localhost:8010 | Out-Null }
```

Browse to [http://localhost:3000](http://localhost:3000)

When the UI loads, you’ll be in your “home” dashboard--click on the Home link at the top left, and you’ll see a list of dashboards.  Click Image Gallery to load the application dashboard.

There are some key data points you need, to make sure you’re monitoring the right things--Google discusses this in the Site Reliability Engineering book [[Google Site Reliability Engineering Book]]. Their focus is on latency, traffic, errors, and saturation, which they call the “golden signals.”

Dockerfile to package custom Grafana image
```dockerfile
FROM diamol/grafana:6.4.3
 
COPY datasource-prometheus.yaml ${GF_PATHS_PROVISIONING}/datasources/
COPY dashboard-provider.yaml ${GF_PATHS_PROVISIONING}/dashboards/
COPY dashboard.json /var/lib/grafana/dashboards/
```

The image starts from a specific version of Grafana and then just copies in a set of [[YAML]] and JSON files. When the container starts, Grafana looks for files in specific folders, and it applies any configuration files it finds. The YAML files set up the Prometheus connection and load any dashboards that are in the `/var/lib/Grafana/dashboards` folder. The final line copies the dashboard JSON into that folder, so it gets loaded when the container starts.

## 9.5 - Understanding the levels of observability
 > Observability is a key requirement when you move from simple proof-of-concept containers to getting ready for production. But there’s another very good reason I introduced Prometheus and Grafana in this chapter: learning Docker is not just about the mechanics of Dockerfiles and Docker Compose files. Part of the magic of Docker is the huge ecosystem that’s grown around containers, and the patterns that have emerged around that ecosystem.
