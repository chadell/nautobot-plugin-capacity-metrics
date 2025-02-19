# nautobot-plugin-capacity-metrics

A plugin for [Nautobot](https://github.com/nautobot/nautobot) to expose additional metrics information.

The plugin is composed of multiple features that can be used independently:
- Application Metrics Endpoint: prometheus endpoint at `/api/plugins/capacity-metrics/app-metrics`
- RQ Queue Metrics Endpoint: prometheus endpoint at `/api/plugins/capacity-metrics/rq-metrics`
- RQ Worker Metrics Command: Add prometheus endpoint on each RQ worker

# Application Metrics Endpoint

Nautobot already exposes some information via a Prometheus endpoint but the information currently available are mostly at the system level and not at the application level.
- **SYSTEM Metrics** are very useful to instrument code, track ephemeral information and get a better visibility into what is happening. (Example of metrics: nbr of requests, requests per second, nbr of exceptions, response time, etc ...) The idea is that when multiple instances of Nautobot are running behind a load balancer each one will produce a different set of metrics and the monitoring system needs to collect these metrics from all running instances and aggregate them in a dashboard. Nautobot exposes some system metrics at `localhost/metrics` [Nautobot DOC](https://nautobot.readthedocs.io/en/stable/additional-features/prometheus-metrics/).
- **APPLICATION Metrics** are at a higher level and represent information that is the same across all instances of an application running behind a load balancer. If I have 3 instances of Nautobot running, there is no point to ask each of them how many Device objects I have in the database, since they will always return the same information. In this case, the goal is to expose only 1 endpoint that can be served by any running instance.

System metrics and application level metrics are complementary with each other

Currently the plugin exposes these simple metrics by default:
- RQ Queues stats
- Jobs stats
- Models count (configurable via configuration.py)

# Queue System Metrics Endpoint

In addition to the default Nautobot system metrics which are exposed at `/metrics` which are largely centered around the Django system on which Nautobot is based this plugin provides some additional system metrics around the queuing system Nuatobot uses to communicate with the Nautobot worker services.  This endpoint is provided separately via `/api/plugins/capacity-metrics/rq-metrics`, this endpoint can be scraped more frequently than the other application metrics endpoint.
## Add your own metrics

This plugin supports some options to generate and publish your own application metrics behind the same endpoint.

### Option 1 - Register function(s) via configuration.py.

It's possible to create your own function to generate some metrics and register it to the plugin in the configuration.py.
Here is an example where the custom function are centralized in a `metrics.py` file, located next to the main `configuration.py`.

```python
# metrics.py
from prometheus_client.core import GaugeMetricFamily

def metric_prefix_utilization():
    """Report prefix utilization as a metric per container."""
    from ipam.models import Prefix  # pylint: disable=import-outside-toplevel

    containers = Prefix.objects.filter(status="container").all()
    g = GaugeMetricFamily(
        "nautobot_prefix_utilization", "percentage of utilization per container prefix", labels=["prefix", "role", "site"]
    )

    for container in containers:

        site = "none"
        role = "none"
        if container.role:
            role = container.role.slug

        if container.site:
            site = container.site.slug

        g.add_metric(
            [str(container.prefix), site, role], container.get_utilization(),
        )

    yield g
```
The new function can be imported in the `configuration.py` file and registered with the plugin.
```python
# configuration.py
from nautobot.metrics import metric_prefix_utilization
PLUGINS_CONFIG = {
    "nautobot_capacity_metrics": {
      "app_metrics": {
        "extras": [
          metric_prefix_utilization
        ]
      }
    }
},
```

### Option 2 - Registry for third party plugins

Any plugin can include its own metrics to improve the visibility and/or the troubleshooting of the plugin itself.
Third party plugins can register their own function(s) using the `ready()` function as part of their PluginConfig class.

```python
# my_plugin/__init__.py
from nautobot_capacity_metrics import register_metric_func
from nautobot.metrics import metric_circuit_bandwidth

class MyPluginConfig(PluginConfig):
    name = "nautobot_myplugin"
    verbose_name = "Demo Plugin "
    # [ ... ]
    def ready(self):
        super().ready()
        register_metric_func(metric_circuit_bandwidth)
```

### Option 3 - NOT AVAILABLE YET - Metrics directory

In the future it will be possible to add metrics by adding them in a predefined directory, similar to jobs.

## Parameters

The behavior of the app_metrics feature can be controlled with the following list of settings (under `nautobot_capacity_metrics > app_metrics`):
- `jobs` boolean (default True), publish stats about the jobs (success, warning, info, failure)
- `queues` boolean (default True), publish stats about RQ Worker (nbr of worker, nbr and type of job in the different queues)
- `models` nested dict, publish the count for a given object (Nbr Device, Nbr IP etc.. ). The first level must be the name of the module in lowercase (dcim, ipam etc..), the second level must be the name of the object (usually starting with a uppercase)
    ```python
    {
      "nautobot.dcim": {"Site": True, "Rack": True, "Device": True,},
      "nautobot.ipam": {"IPAddress": True, "Prefix": True}
    }
    ```
## Usage

Configure your Prometheus server to collect the application metrics at `/api/plugins/capacity-metrics/app-metrics/`

```yaml
# Sample prometheus configuration
scrape_configs:
  - job_name: 'nautobot_app'
    scrape_interval: 120s
    metrics_path: /api/plugins/capacity-metrics/app-metrics
    static_configs:
      - targets: ['nautobot']
  - job_name: 'nautobot_queue'
    scrape_interval: 20s
    metrics_path: /api/plugins/capacity-metrics/rq-metrics
    static_configs:
      - targets: ['nautobot']
```

# RQ Worker Metrics Endpoint

This plugin add a new django management command `rqworker_metrics` that is behaving identically to the default `rqworker` command except that this command also exposes a prometheus endpoint (default port 8001).

With this endpoint it become possible to instrument the tasks running asyncronously in the worker.

## Usage

The new command needs to be executed on the worker as a replacement for the default `rqworker`
```
nautobot-server rqworker_metrics
```

The port used to expose the prometheus endpoint can be configured for each worker in CLI.
```
nautobot-server rqworker_metrics --prom-port 8002
```

Since the rq-worker is based on a fork model, for this feature to work it''s required to use prometheus in multi processes mode.
To enable this mode the environment variable `prometheus_multiproc_dir` must be define and point at a valid directory.

# Installation

The plugin is available as a Python package in pypi and can be installed with pip
```shell
pip install nautobot-plugin-capacity-metrics
```

> The plugin is compatible with Nautobot 2.8.1 and higher

To ensure Application Metrics Plugin is automatically re-installed during future upgrades, create a file named `local_requirements.txt` (if not already existing) in the Nautobot root directory (alongside `requirements.txt`) and list the `nautobot-plugin-capacity-metrics` package:

```no-highlight
# echo nautobot-plugin-capacity-metrics >> local_requirements.txt
```

Once installed, the plugin needs to be enabled in your `configuration.py`
```python
# In your configuration.py
PLUGINS = ["nautobot_capacity_metrics"]

# PLUGINS_CONFIG = {
#   "nautobot_capacity_metrics": {
#     "app_metrics": {
#       "models": {
#          "nautobot.dcim": {"Site": True, "Rack": True, "Device": True,},
#          "nautobot.ipam": {"IPAddress": True, "Prefix": True},
#        },
#        "jobs": True,
#        "queues": True,
#       }
#     }
#   }
# }
```

## Included Grafana Dashboard

Included within this plugin is a Grafana dashboard which will work with the example configuration above. To install this dashboard import the JSON from [Grafana Dashboard](nautobot_grafana_dashboard.json) into Grafana.

![Nautobot Grafana Dashboard](nautobot_grafana_dashboard.png)

# Contributing

Pull requests are welcomed and automatically built and tested against multiple version of Python and multiple version of Nautobot through TravisCI.

The project is packaged with a light development environment based on `docker-compose` to help with the local development of the project and to run the tests within TravisCI.

The project is following Network to Code software development guideline and is leveraging:
- Black, Pylint, Bandit and pydocstyle for Python linting and formatting.
- Django unit test to ensure the plugin is working properly.

### CLI Helper Commands

The project is coming with a CLI helper based on [invoke](http://www.pyinvoke.org/) to help setup the development environment. The commands are listed below in 3 categories `dev environment`, `utility` and `testing`.

Each command can be executed with `invoke <command>`. All commands support the arguments `--nautobot-ver` and `--python-ver` if you want to manually define the version of Python and Nautobot to use. Each command also has its own help `invoke <command> --help`

#### Local dev environment
```
  build            Build all docker images.
  debug            Start Nautobot and its dependencies in debug mode.
  destroy          Destroy all containers and volumes.
  start            Start Nautobot and its dependencies in detached mode.
  stop             Stop Nautobot and its dependencies.
```

#### Utility
```
  cli              Launch a bash shell inside the running Nautobot container.
  create-user      Create a new user in django (default: admin), will prompt for password.
  makemigrations   Run Make Migration in Django.
  nbshell          Launch a nbshell session.
```
#### Testing

```
  tests            Run all tests for this plugin.
  pylint           Run pylint code analysis.
  pydocstyle       Run pydocstyle to validate docstring formatting adheres to NTC defined standards.
  bandit           Run bandit to validate basic static code security analysis.
  black            Run black to check that Python files adhere to its style standards.
  unittest         Run Django unit tests for the plugin.
```

## Questions

For any questions or comments, please check the [FAQ](FAQ.md) first and feel free to swing by the [Network to Code slack channel](https://networktocode.slack.com/) (channel #networktocode).
Sign up [here](http://slack.networktocode.com/)

## Default Metrics for the application metrics endpoint

The following metrics will be provided via the `/api/plugins/capacity-metrics/app-metrics` endpoint:

```
# HELP nautobot_job_task_stats Per Job task statistics
# TYPE nautobot_job_task_stats gauge
nautobot_job_task_stats{module="local/users/CheckUser",name="total",status="success"} 1.0
nautobot_job_task_stats{module="local/users/CheckUser",name="total",status="warning"} 0.0
nautobot_job_task_stats{module="local/users/CheckUser",name="total",status="failure"} 0.0
nautobot_job_task_stats{module="local/users/CheckUser",name="total",status="info"} 0.0
nautobot_job_task_stats{module="local/users/CheckUser",name="test_is_uppercase",status="success"} 1.0
nautobot_job_task_stats{module="local/users/CheckUser",name="test_is_uppercase",status="warning"} 0.0
nautobot_job_task_stats{module="local/users/CheckUser",name="test_is_uppercase",status="failure"} 0.0
nautobot_job_task_stats{module="local/users/CheckUser",name="test_is_uppercase",status="info"} 0.0
# HELP nautobot_job_execution_status Job completion status
# TYPE nautobot_job_execution_status gauge
nautobot_job_execution_status{module="local/users/CheckUser",status="pending"} 0.0
nautobot_job_execution_status{module="local/users/CheckUser",status="running"} 0.0
nautobot_job_execution_status{module="local/users/CheckUser",status="completed"} 1.0
nautobot_job_execution_status{module="local/users/CheckUser",status="errored"} 0.0
nautobot_job_execution_status{module="local/users/CheckUser",status="failed"} 0.0
# HELP nautobot_model_count Per Nautobot Model count
# TYPE nautobot_model_count gauge
nautobot_model_count{app="dcim",name="Site"} 24.0
nautobot_model_count{app="dcim",name="Rack"} 24.0
nautobot_model_count{app="dcim",name="Device"} 46.0
nautobot_model_count{app="ipam",name="IPAddress"} 58.0
nautobot_model_count{app="ipam",name="Prefix"} 18.0
# HELP nautobot_app_metrics_processing_ms Time in ms to generate the app metrics endpoint
# TYPE nautobot_app_metrics_processing_ms gauge
nautobot_app_metrics_processing_ms 59.48257
```

The following metrics will be provided via the `/api/plugins/capacity-metrics/rq-metrics` endpoint:

```
# HELP nautobot_queue_number_jobs Number of Job per RQ queue and status
# TYPE nautobot_queue_number_jobs gauge
nautobot_queue_number_jobs{name="check_releases",status="finished"} 0.0
nautobot_queue_number_jobs{name="check_releases",status="started"} 0.0
nautobot_queue_number_jobs{name="check_releases",status="deferred"} 0.0
nautobot_queue_number_jobs{name="check_releases",status="failed"} 0.0
nautobot_queue_number_jobs{name="check_releases",status="scheduled"} 0.0
nautobot_queue_number_jobs{name="default",status="finished"} 0.0
nautobot_queue_number_jobs{name="default",status="started"} 0.0
nautobot_queue_number_jobs{name="default",status="deferred"} 0.0
nautobot_queue_number_jobs{name="default",status="failed"} 0.0
nautobot_queue_number_jobs{name="default",status="scheduled"} 0.0
# HELP nautobot_queue_number_workers Number of worker per queue
# TYPE nautobot_queue_number_workers gauge
nautobot_queue_number_workers{name="check_releases"} 0.0
nautobot_queue_number_workers{name="default"} 2.0
# HELP nautobot_rq_metrics_processing_ms Time in ms to generate the app metrics endpoint
# TYPE nautobot_rq_metrics_processing_ms gauge
nautobot_rq_metrics_processing_ms 33.34308
```