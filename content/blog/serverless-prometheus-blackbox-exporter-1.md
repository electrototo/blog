+++
title = 'Serverless Prometheus Blackbox Exporter pt 1'
description = 'Porting prometheus blackbox exporter to an AWS lambda using custom lambda environments, and a little bit of source code modification.'
date = 2024-09-14T15:02:32-07:00
draft = true
tags = ["Lambda"]
categories = ["Observability"]
[params]
    [params.author]
        email = 'cristobal@liendo.net'
        name = 'Cristobal Liendo'
+++

Recently I discovered [Prometheus](https://prometheus.io/) which is a monitoring system that collects metrics from different data sources by polling
on certain cadence. Afterwards you can access and query the metrics that were obtained through different interfaces such as [Grafana](https://grafana.com/) and
a web UI.

Being a monitoring tool that is capable to obtain metrics such as latency and availability of services, I started to wonder if I can deploy it in AWS Lambda, mainly for two
reasons:
1. Lambda is relatively cheap.
2. Lambda is avaialble in many, many AWS regions around the world. 

Point number 2 interests me, as I can theoretically monitor a service (for example this blog) from different geographical places and have a basic understanding regarding how is it performing.

## Multi-Target Exporter Pattern
I want to focus on the [Multi-Target Exporter Pattern](https://prometheus.io/docs/guides/multi-target-exporter/) that is common within Prometheus. As high level,
this exporter pattern is used to obtain metrics related to latency and reachability outside from the network that prometheus is running on.

A sample request to a local installation of the [blackbox_exporter](https://github.com/prometheus/blackbox_exporter), which implements the multi-target exporter pattern
looks like the following:

### Request
The request is reaching the blackbox exporter running at the port 9115 of my laptop, and it is requesting to probe [cristobal.liendo.net](https://cristobal.liendo.net) with the
module `http_2xx`.

```bash
curl 'localhost:9115/probe?target=cristobal.liendo.net&module=http_2xx' > sample-output.txt
```

I will not dive into the configuration of blackbox exporter, but basically modules indicate _how_ should the probe work: https configuration, timeouts, assertions, among others. Depending
on the configuration of the module, the response containing the metrics will vary.


### Response
```
# HELP probe_dns_lookup_time_seconds Returns the time taken for probe dns lookup in seconds
# TYPE probe_dns_lookup_time_seconds gauge
probe_dns_lookup_time_seconds 0.118188376
# HELP probe_duration_seconds Returns how long the probe took to complete in seconds
# TYPE probe_duration_seconds gauge
probe_duration_seconds 0.270411918
# HELP probe_failed_due_to_regex Indicates if probe failed due to regex
# TYPE probe_failed_due_to_regex gauge
probe_failed_due_to_regex 0
# HELP probe_http_content_length Length of http content response
# TYPE probe_http_content_length gauge
probe_http_content_length 2078
# HELP probe_http_duration_seconds Duration of http request by phase, summed over all redirects
# TYPE probe_http_duration_seconds gauge
probe_http_duration_seconds{phase="connect"} 0.005934780000000001
probe_http_duration_seconds{phase="processing"} 0.091509201
probe_http_duration_seconds{phase="resolve"} 0.120317337
probe_http_duration_seconds{phase="tls"} 0.050705509
probe_http_duration_seconds{phase="transfer"} 0.000334677
# HELP probe_http_last_modified_timestamp_seconds Returns the Last-Modified HTTP response header in unixtime
# TYPE probe_http_last_modified_timestamp_seconds gauge
probe_http_last_modified_timestamp_seconds 1.726642887e+09
... omitted for brevity ...
```

Voilà! I have latency metrics for DNS lookup time, total duration to reach the website, and more granular duration metrics.

Keep in mind that these values are perceived from the place where the blackbox exporter server was running, in this case: my laptop.

## Porting blackbox exporter to AWS Lambda
As high-level we need to do two things to port blackbox exporter to Lambda, which basically will merge into a single file:
1. Use a custom AWS Lambda Runtime Environment[^1] in which the blackbox binary will be running.
2. Create a _wrapper_ between lambda and the blackbox process.

All the code that will be used on this section can be found in my GitHub repository: [aws-lambda-blackbox-exporter](https://github.com/electrototo/aws-lambda-blackbox-exporter).

### Custom AWS Lambda Runtime Environment

### Blackbox exporter wrapper

[^1]: If you want to learn more about custom AWS Lambda Runtime Environments, feel free to read my [previous blog post]({{< ref "custom-aws-lambda-runtime-environments" >}})!