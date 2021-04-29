# Lesson 8: Introduction to Checks

- [Overview](#overview)
- [Scheduling](#scheduling)
- [Subscriptions](#subscriptions)
- [Check templates](#check-templates)
- [Metrics collection](#metrics-collection)
  - [Output Metric Extraction](#output-metric-extraction)
  - [Output Metric Handlers](#output-metric-handlers)
  - [Output Metric Tags](#output-metric-tags)
- [Advanced Topics](#advanced-topics)
  - [TTLs (Dead Man Switches)](#ttls-dead-man-switches)
  - [Proxy checks (pollers)](#proxy-checks-pollers)
  - [Execution environment & environment variables](#execution-environment--environment-variables)
- [EXERCISE 1: configure a check](#exercise-1-configure-a-check)
- [EXERCISE 2: modify a check configuration using tokens](#exercise-2-modify-a-check-using-tokens)
- [EXERCISE 3: collecting metrics with Sensu Checks](#exercise-3-collecting-metrics-with-sensu-checks)
- [Next steps](#next-steps)
- [Learn more](#learn-more)

## Overview

Sensu Checks are monitoring jobs that are managed by the Sensu platform (control plane) and executed by Sensu Agents.
A Sensu Check is a modern take on the traditional "service check" &ndash; a task (check) performed by the monitoring platform to determine the status of a system or service.

<details>
<summary><strong>Example monitoring job (check) configuration template:</strong></summary>

```yaml
type: CheckConfig
api_version: core/v2
metadata:
  name: node_exporter
spec:
  command: wget -q -O- http://127.0.0.1:{{ .labels.node_exporter_port | default "9100" }}/metrics
  runtime_assets: []
  publish: true
  interval: 30
  subscriptions:
  - linux
  timeout: 10
  ttl: 0
  output_metric_format: prometheus_text
  output_metric_handlers:
  - elasticsearch
  output_metric_tags:
  - name: entity
    value: "{{ .name }}"
  - name: region
    value: "{{ .labels.region | default 'unknown' }}"
```

</details>

Although service checks were originally popularized by Nagios (circa 1999-2002), they continue to fill a critical role in the modern era of cloud computing.
Sensu orchestrates service checks in a similar manner as cloud-native platforms like Kubernetes and Prometheus which use "Jobs" as a central concept for scheduling and running tasks.
Where Prometheus jobs are limited to HTTP GET requests (for good reason), a Sensu monitoring job ("check") provides a significantly more flexible tool.

A valid service check must satisfy the following requirements:

1. Communicate status via exit status codes
1. Emit service status information and telemetry data via STDOUT

That's the entire specification (more or less)!
Service checks have provided sustained value thanks to this incredibly simple specification, providing tremendous extensibility.
In fact, service checks can be written in any programming language in the world (including simple Bash and MS DOS scripts).

## Scheduling

The Sensu backend handles the scheduling of all monitoring jobs (checks).
Check scheduling is configured using the following attributes:

- **`publish`:** enables or disables scheduling
- **`interval` or `cron`:** the actual schedule upon which check requests will be published to the corresponding subscriptions
- **`subscriptions`:** the subscriptions to publish check requests to
- **`round_robin`:** limits check scheduling to one execution per request (useful for configuring pollers when there are multiple agent members in a given subscription)
- **`timeout`:** instructs the agent to terminate check execution after the configured number of seconds

## Subscriptions

As discussed in [Lesson 7](/lessons/operator/07/README.md#subscriptions), Sensu uses the [publish/subscribe model of communication](https://en.wikipedia.org/wiki/Publish–subscribe_pattern).
Sensu schedules monitoring jobs (checks) at a pre-set intervals, automatically "publishing" requests to the configured topics (subscriptions).

Because subscriptions are [loosely coupled](https://en.wikipedia.org/wiki/Loose_coupling) references, Sensu checks can be configured with subscriptions that have no agent members and the result is simply a ["no-op"](https://en.wikipedia.org/wiki/NOP_(code)) (no action is taken).
This works especially well in ephemeral or elastic infrastructures where host-based monitoring configuration is ineffective.
Instead of configuring monitoring on a per-host basis, monitoring configuration can be predefined following a service-based model (e.g. with one subscription per service, such as "postgres"), and agents on ephemeral compute instances simply register with a Sensu backend, subscribe to to the relevant monitoring "topics" and begin reporting observability data.

## Check templates

Monitoring jobs (checks) can be templated using placeholders called "Sensu Tokens" which are replaced with entity information before the job is executed.
Token substitution is performed by the Sensu Agent<sup>1</sup>, during which all tokens are replaced with the corresponding entity data prior to check execution.
Sensu Tokens are available for Checks, Hooks (see [Lesson 9: Introduction to Check Hooks](/lessons/operator/09/README.md#readme)), and Assets (see [Lesson 10: Introduction to Assets](/lessons/operator/10/README.md#readme)).

Sensu Tokens are references to entity attributes and metadata, wrapped in double curly braces (`{{  }}`).
Default values can also be provided as a fallback for [unmatched tokens](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/tokens/#unmatched-tokens).
Sensu Tokens can be used to configure dynamic monitoring jobs (e.g. enabling node-based configuration overrides for things like alerting threshold, etc).

**Examples:**

- **`{{ .name }}`:** replaced by the target entity name
- **`{{ .labels.url }}`:** replaced by the target entity "url" label
- **`{{ .labels.disk_warning | default "85%" }}`:** replaced by the target entity "disk_warning" label; if the label is not set then the default/fallback value of `85%` will be used

==TODO: add an example check config w/ .labels.disk_warning label==

<sup>1: Token substution is performed by the Sensu Agent for standard checks only.
Token substitution is performed by the Sensu Backend for [proxy checks](#proxy-checks-pollers).</sup>

## Metrics collection

One popular use case for monitoring jobs (checks) is to collect various system and service metrics (e.g. cpu, memory, or disk utilization; or api response times).

To learn more about Sensu metrics processing capabilities, please visit the [Sensu Metrics reference documentation](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/metrics/).

### Output Metric Extraction

The Sensu Agent provides built-in support for normalizing metrics generated by service checks in the following `output_metric_format`s:

- **`prometheus_text`:** [Prometheus exposition format](https://prometheus.io/docs/instrumenting/exposition_formats/)
- **`influxdb_line`:** [InfluxDB line protocol](https://docs.influxdata.com/influxdb/cloud/reference/syntax/line-protocol/)
- **`opentsdb_line`:** [OpenTSDB line protocol](http://opentsdb.net/docs/build/html/user_guide/writing/index.html)
- **`graphite_plaintext`:** [Graphite plaintext protocol](https://graphite.readthedocs.io/en/latest/feeding-carbon.html)
- **`nagios_perfdata`:** [Nagios Performance Data](https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/perfdata.html)

Configuring `output_metrics` causes the agent to extract metrics at the edge &ndash; before sending event data to the observability pipeline &ndash; optimizing performance of the platform at scale.

> **NOTE:** Sensu also provides support for collecting StatsD metrics, however these are consumed via the StatsD API &ndash; not collected as output of a monitoring job (check).

### Output Metric Handlers

In addition to `output_metric_format`, Sensu checks also provide configuration for dedicated `output_metric_handlers` &ndash; event handlers that are specially optimized for processing metrics (only).
If an event containing metrics is configured with one or more `output_metric_handlers`, a copy of the event is forwarded to the metric handler prior to Sensu's own event persistence; this specialized handling is implemented as a performance optimization to prioritize metric processing.

> **NOTE:** Sensu checks may be configured with one or more `handlers` and `output_metric_handlers`, enabling service health checking and alerting _and_ metrics collection in a single monitoring job.

### Output Metric Tags

Metrics extracted with `output_metrics_format` can also be enriched using `output_metric_tags`.
Metric sources vary in verbosity &ndash; some metric formats don't support tags (e.g. Nagios Performance Data), and even those that do can be implemented in ways that simply don't provide enough contextual data.
In either case, Sensu's `output_metric_tags` are great for enriching collected metrics using entity data/metadata.
Sensu breathes new life into legacy monitoring plugins or other metric sources that generate the raw data you care about, but lack tags or other context to make sense of the data; simply configure `output_metric_tags` and Sensu will add the corresponding tag data to the resulting metrics/measurements.

**Example:**

```yaml
output_metric_tags:
- name: application
  value: "my-app"
- name: entity
  value: "{{ .name }}"
- name: region
  value: "{{ .labels.region | default 'unknown' }}"
- name: store_id
  value: "store/{{ .labels.store_id | default 'none' }}"
```

Metric tag values can be provided as strings, or [Sensu Tokens](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/tokens/) which can be used for generating dynamic tag values.

## Advanced Topics

### TTLs (Dead Man Switches)

### Proxy Checks (Pollers)

The Sensu check scheduler can orchestrate monitoring jobs for entities that are not actively managed by a Sensu agent.
These monitoring jobs are called "proxy checks", or checks that target a proxy entity.
Proxy checks are discussed in greater detail in [Lesson 13: Introduction to Proxy Entities & Proxy Checks](/lessons/operator/13/README.md#readme).

At a high level, a proxy check is a Sensu check with `proxy_requests`, which are effectively query parameters Sensu will use to look for matching entities that should be targeted by the check.
Proxy requests are published to the configured subscription(s) once per matching entity.
In the following example, we would expect Sensu to find two (2) entities with `entity_class == "proxy"` and a `proxy_type` label set to "website"; for each matching entity, the Sensu backend will first replace the configured tokens using the corresponding entity attributes (i.e. one request to execute the command `nslookup sensu.io`, and one request to execute the command `nslookup google.com`).
To avoid redundant processing, we recommend using the `round_robin` attribute with proxy checks.

```yaml
---
type: CheckConfig
api_version: core/v2
metadata:
  name: proxy-nslookup
spec:
  command: >-
    nslookup {{ .annotations.proxy_host }}
  runtime_assets: []
  publish: true
  subscriptions:
  - workshop
  interval: 30
  timeout: 10
  round_robin: true
  proxy_requests:
    entity_attributes:
      - entity.entity_class == "proxy"
      - entity.labels.proxy_type == "website"

---
type: Entity
api_version: core/v2
metadata:
  name: proxy-a
  labels:
    proxy_type: website
  annotations:
    proxy_host: sensu.io
spec:
  entity_class: proxy

---
type: Entity
api_version: core/v2
metadata:
  name: proxy-b
  labels:
    proxy_type: website
  annotations:
    proxy_host: google.com
spec:
  entity_class: proxy
```

### Execution environment & environment variables

## EXERCISE 1: configure a check

1. **Configure a Sensu Check for monitoring disk usage.**

   Copy and paste the following contents to a file named `disk.yaml`:

   ```yaml
   ---
   type: CheckConfig
   api_version: core/v2
   metadata:
     name: disk
   spec:
     command: check-disk-usage --warning 80.0 --critical 90.0
     runtime_assets:
     - sensu/check-disk-usage:0.4.2
     publish: true
     interval: 30
     subscriptions:
     - system/macos
     - system/macos/disk
     - system/windows
     - system/windows/disk
     - system/linux
     - system/linux/disk
     timeout: 10
     check_hooks: []
   ```

   Notice the values of `subscriptions` and `interval` – these will instruct the Sensu platform to schedule (or "publish") monitoring jobs every 30 seconds on any agent with the `system/macos`, `system/windows`, or `system/linux` subscriptions.
   Agents opt-in (or "subscribe") to monitoring jobs by their corresponding `subscriptions` configuration.

1. **Create the Check using the `sensuctl create -f` command.**

   ```shell
   sensuctl create -f disk.yaml
   ```

   Verify that the Check was successfully created using the `sensuctl check list` command:

   ```shell
   sensuctl check list
   ```

   Example output:

   ```shell
     Name                       Command                       Interval   Cron   Timeout   TTL                                            Subscriptions                                             Handlers              Assets              Hooks   Publish?   Stdin?   Metric Format   Metric Handlers
    ────── ───────────────────────────────────────────────── ────────── ────── ───────── ───── ────────────────────────────────────────────────────────────────────────────────────────────────── ────────── ────────────────────────────── ─────── ────────── ─────────────────────── ─────────────────
     disk   check-disk-usage --warning 80.0 --critical 90.0         30               10     0   system/macos,system/macos/disk,system/windows,system/windows/disk,system/linux,system/linux/disk              sensu/check-disk-usage:0.4.2           true       false
   ```

**NEXT:** do you see the `disk` check in the output?
If so, you're ready to move on to the next exercise!

## EXERCISE 2: modify check configuration using tokens

Sensu's service-oriented configuration model (as opposed to traditional host-based models) makes monitoring configuration easier to manage at scale.
A single check definition can be used to collect monitoring data from hundreds or thousands of endpoints!
However, there are often cases when you need to override various monitoring job configuration parameters on an per-endpoint basis.
For these situations, Sensu provides a templating feature called [Tokens](#).

Let's modify our check from the previous exercise using some Tokens.

1. **Update the `disk` check configuration template.**

   Modify `disk.yaml` with the following contents:

   ```yaml
   ---
   type: CheckConfig
   api_version: core/v2
   metadata:
     name: disk-usage
   spec:
     command: >-
       check-disk-usage
       --warning {{ .annotations.disk_usage_warning_threshold | default "80.0" }}
       --critical {{ .annotations.disk_usage_critical_threshold | default "90.0" }}
     runtime_assets:
     - sensu/check-disk-usage:0.4.2
     publish: true
     interval: 30
     subscriptions:
     - system/macos
     - system/macos/disk
     - system/windows
     - system/windows/disk
     - system/linux
     - system/linux/disk
     timeout: 10
     check_hooks: []
   ```

   _NOTE: this example uses a [YAML multiline "block scalar"](https://yaml-multiline.info) (`>-`) for improved readability of a longer check `command` (without the need to escape newlines)._

   Did you notice?
   We're now making the disk usage warning and critical thresholds configurable via entity annotations (`disk_usage_warning_threshold` and `disk_usage_critical_threshold`)!
   Both of the tokens we're using here are offering default values, which will be used if the corresponding annotation is not set.

1. **Update the Check using `sensuctl create -f`.**

   ```
   sensuctl create -f disk.yaml
   ```

   Verify that the Check was successfully created using the `sensuctl check list` command:

   ```shell
   sensuctl check info disk-usage --format yaml
   ```

## EXERCISE 3: collecting metrics with Sensu Checks

1. **Update the `disk` check configuration template.**

   Modify `disk.yaml` with the following contents (adding `output_metric_format`, `output_metric_handlers`, and `output_metric_tags` fields):

   ==TODO==

   These fields instruct Sensu what metric format to expect as output from the check, which handler(s) should be used to process the metrics, and what tags should be added to the metrics.
   The metric formats Sensu can extract from check output as of this writing are: `nagios_perfdata`, `graphite_plaintext`, `influxdb_line`, `opentsdb_line`, and `prometheus_text` (StatsD metrics are also supported, but only via the Sensu Agent StatsD API).

1. **Update the Check using `sensuctl create -f`.**

   ```shell
   sensuctl create -f disk.yaml
   ```

   Verify that the Check was successfully created using the `sensuctl check list` command:

   ```shell
   sensuctl check info disk-usage --format yaml
   ```

## Learn more

- [[Documentation] "Schedule observability data collection" (docs.sensu.io)](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/)
- [[Documentation] "Sensu Checks Reference" (docs.sensu.io)](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/checks/)
- [[Documentation] "Sensu Tokens Reference" (docs.sensu.io)](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/tokens/)
- [[Documentation] "Guide: Monitor server resources with Sensu Checks" (docs.sensu.io)](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/monitor-server-resources/)
- [[Documentation] "Guide: Collect service metrics with Sensu Checks" (docs.sensu.io)](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/collect-metrics-with-checks/)
- [[Documentation] "Guide: Collect Prometheus metrics with Sensu" (docs.sensu.io)](https://docs.sensu.io/sensu-go/latest/observability-pipeline/observe-schedule/prometheus-metrics/)
- [[Blog Post] "Self-service monitoring checks in Sensu Go" (sensu.io)](https://sensu.io/blog/self-service-monitoring-checks-in-sensu-go)
- [[Blog Post] "The story of Nagios plugin support in Sensu (or, why service checks are so amazing)" (sensu.io)](https://sensu.io/blog/the-story-of-nagios-plugin-support-in-sensu)
- [[Blog Post] "Check output metric extraction with InfluxDB & Grafana" (sensu.io)](https://sensu.io/blog/check-output-metric-extraction-with-influxdb-grafana)
- [[Blog Post] "How to collect Prometheus metrics and store them anywhere (with Sensu!)" (sensu.io)](https://sensu.io/blog/how-to-collect-prometheus-metrics-and-store-them-anywhere-with-sensu)

## Next steps

[Share your feedback on Lesson 08](https://github.com/sensu/sensu-go-workshop/issues/new?template=lesson_feedback.md&labels=feedback%2Clesson-08&title=Lesson%2008%20Feedback)

[Lesson 9: Introduction to Check Hooks](../09/README.md#readme)
