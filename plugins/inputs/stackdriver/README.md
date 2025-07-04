# Stackdriver Google Cloud Monitoring Input Plugin

This plugin collects metrics from [Google Cloud Monitoring][gcm]
(formerly Stackdriver) using the [Cloud Monitoring API v3][stackdriver].

> [!IMPORTANT]
> This plugin accesses APIs which are [chargeable][pricing], cost might incur.

⭐ Telegraf v1.10.0
🏷️ cloud
💻 all

[gcm]: https://cloud.google.com/monitoring
[stackdriver]: https://cloud.google.com/monitoring/api/v3/
[pricing]: https://cloud.google.com/stackdriver/pricing#stackdriver_monitoring_services

## Global configuration options <!-- @/docs/includes/plugin_config.md -->

In addition to the plugin-specific configuration settings, plugins support
additional global and plugin configuration settings. These settings are used to
modify metrics, tags, and field or create aliases and configure ordering, etc.
See the [CONFIGURATION.md][CONFIGURATION.md] for more details.

[CONFIGURATION.md]: ../../../docs/CONFIGURATION.md#plugins

## Configuration

```toml @sample.conf
# Gather timeseries from Google Cloud Platform v3 monitoring API
[[inputs.stackdriver]]
  ## GCP Project
  project = "erudite-bloom-151019"

  ## Include timeseries that start with the given metric type.
  metric_type_prefix_include = [
    "compute.googleapis.com/",
  ]

  ## Exclude timeseries that start with the given metric type.
  # metric_type_prefix_exclude = []

  ## Most metrics are updated no more than once per minute; it is recommended
  ## to override the agent level interval with a value of 1m or greater.
  interval = "1m"

  ## Maximum number of API calls to make per second.  The quota for accounts
  ## varies, it can be viewed on the API dashboard:
  ##   https://cloud.google.com/monitoring/quotas#quotas_and_limits
  # rate_limit = 14

  ## The delay and window options control the number of points selected on
  ## each gather.  When set, metrics are gathered between:
  ##   start: now() - delay - window
  ##   end:   now() - delay
  #
  ## Collection delay; if set too low metrics may not yet be available.
  # delay = "5m"
  #
  ## If unset, the window will start at 1m and be updated dynamically to span
  ## the time between calls (approximately the length of the plugin interval).
  # window = "1m"

  ## TTL for cached list of metric types.  This is the maximum amount of time
  ## it may take to discover new metrics.
  # cache_ttl = "1h"

  ## If true, raw bucket counts are collected for distribution value types.
  ## For a more lightweight collection, you may wish to disable and use
  ## distribution_aggregation_aligners instead.
  # gather_raw_distribution_buckets = true

  ## Aggregate functions to be used for metrics whose value type is
  ## distribution.  These aggregate values are recorded in in addition to raw
  ## bucket counts; if they are enabled.
  ##
  ## For a list of aligner strings see:
  ##   https://cloud.google.com/monitoring/api/ref_v3/rpc/google.monitoring.v3#aligner
  # distribution_aggregation_aligners = [
  #  "ALIGN_PERCENTILE_99",
  #  "ALIGN_PERCENTILE_95",
  #  "ALIGN_PERCENTILE_50",
  # ]

  ## Filters can be added to reduce the number of time series matched.  All
  ## functions are supported: starts_with, ends_with, has_substring, and
  ## one_of.  Only the '=' operator is supported.
  ##
  ## The logical operators when combining filters are defined statically using
  ## the following values:
  ##   filter ::= <resource_labels> {AND <metric_labels> AND <user_labels> AND <system_labels>}
  ##   resource_labels ::= <resource_labels> {OR <resource_label>}
  ##   metric_labels ::= <metric_labels> {OR <metric_label>}
  ##   user_labels ::= <user_labels> {OR <user_label>}
  ##   system_labels ::= <system_labels> {OR <system_label>}
  ##
  ## For more details, see https://cloud.google.com/monitoring/api/v3/filters
  #
  ## Resource labels refine the time series selection with the following expression:
  ##   resource.labels.<key> = <value>
  # [[inputs.stackdriver.filter.resource_labels]]
  #   key = "instance_name"
  #   value = 'starts_with("localhost")'
  #
  ## Metric labels refine the time series selection with the following expression:
  ##   metric.labels.<key> = <value>
  #  [[inputs.stackdriver.filter.metric_labels]]
  #    key = "device_name"
  #    value = 'one_of("sda", "sdb")'
  #
  ## User labels refine the time series selection with the following expression:
  ##   metadata.user_labels."<key>" = <value>
  #  [[inputs.stackdriver.filter.user_labels]]
  #    key = "environment"
  #    value = 'one_of("prod", "staging")'
  #
  ## System labels refine the time series selection with the following expression:
  ##   metadata.system_labels."<key>" = <value>
  #  [[inputs.stackdriver.filter.system_labels]]
  #    key = "machine_type"
  #    value = 'starts_with("e2-")'
```

### Authentication

It is recommended to use a service account to authenticate with the
Stackdriver Monitoring API.  [Getting Started with Authentication][auth].

[auth]: https://cloud.google.com/docs/authentication/getting-started

## Troubleshooting

When Telegraf is ran with `--debug`, detailed information about the performed
queries will be logged.

## Metrics

Metrics are created using one of there patterns depending on if the value type
is a scalar value, raw distribution buckets, or aligned bucket values.

In all cases, the Stackdriver metric type is split on the last component into
the measurement and field:

```sh
compute.googleapis.com/instance/disk/read_bytes_count
└──────────  measurement  ─────────┘ └──  field  ───┘
```

### Scalar Values

- measurement
  - tags:
    - resource_labels
    - metric_labels
  - fields:
    - field

### Distributions

Distributions are represented by a set of fields along with the bucket values
tagged with the bucket boundary.  Buckets are cumulative: each bucket
represents the total number of items less than the `lt` tag.

- measurement
  - tags:
    - resource_labels
    - metric_labels
  - fields:
    - field_count
    - field_mean
    - field_sum_of_squared_deviation
    - field_range_min
    - field_range_max

- measurement
  - tags:
    - resource_labels
    - metric_labels
    - lt (less than)
  - fields:
    - field_bucket

### Aligned Aggregations

- measurement
  - tags:
    - resource_labels
    - metric_labels
  - fields:
    - field_alignment_function

## Example Output

The output depends on the defined filters and metric types.
