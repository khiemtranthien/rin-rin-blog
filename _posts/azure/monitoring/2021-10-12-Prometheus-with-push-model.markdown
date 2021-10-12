---
layout: post
title:  "Prometheus with Push model"
date:   2021-10-12 21:13:37 +0700
categories: azure-cloud
tags: prometheus aks
---

Prometheus comes with two models to ingest data. Pull and Push model.  
In **pull** model, Prometheus will call your app to scrape metric data. Your app need to publish /metrics endpoint. This way is used for long-live apps.  

In **push** model, you actively push data to Prometheus through Push Gateway. Prometheus will connect to Push Gateway to scrape your metric data. This way is used for batch jobs and short-live app.  

## Send data to Prometheus via Push Gateway

By default, when you setup Prometheus with Helm chart, it already installed Push Gateway with IP is the ClusterIP in service. We are going to change it to LoadBalancer.

### Configure Push Gateway

In prometheus.values.ymal file, navigate to pushgateway > service, change service type to LoadBalancer.
```
loadBalancerIP: ""
loadBalancerSourceRanges: []
servicePort: 9091
type: LoadBalancer
```
Now, we can see the metric data in Push Gateway by visit: http://push-gateway-external-ip/metrics

Go to serverFiles > prometheus.yml > scrape_configs, add a new job to get metric data from Push Gateway.
```
- job_name: push-gateway
        static_configs:
          - targets:
              - my-prometheus-pushgateway:9091
        honor_labels: true
```

Apply your changes to AKS:
```
helm upgrade my-prometheus prometheus-community/prometheus --set server.service.type=LoadBalancer --set rbac.create=false -f prometheus.values.yaml
```

### Send data from batch job

In your batch job, install two Nuget packages:
```
prometheus-net
prometheus-net.AspNetCore
```

Create Pusher instance to sync data to Push Gateway:
```
Counter counter = Metrics.CreateCounter("push_counter", "Request counter from Push gateway");

void OnError(Exception ex)
{
    System.Console.WriteLine($"{ex.Message} - {ex.StackTrace}");
}

[HttpGet]
public IEnumerable<WeatherForecast> Get()
{
    var pusher = new MetricPusher(new MetricPusherOptions
    {
        Endpoint = "http://push-gateway-external-ip:9091/metrics",
        IntervalMilliseconds = 100,
        Job = "your-job-name",
        OnError = OnError
    });

    pusher.Start();

    int idx = 0;
    while(idx < 1000)
    {
        counter.Inc();
        Thread.Sleep(1000);
        System.Console.WriteLine($"Increase by 1. Counter {idx}");

        idx++;
    }

    pusher.Stop();

    var rng = new Random();
    return Enumerable.Range(1, 5).Select(index => new WeatherForecast
    {
        Date = DateTime.Now.AddDays(index),
        TemperatureC = rng.Next(-20, 55),
        Summary = Summaries[rng.Next(Summaries.Length)]
    })
    .ToArray();
}
```

You are able see your counter change over time (every 2 minutes in this case) with the query in Prometheus page:
```
increase(push_counter{job="your-job-name"}[2m]
```

## Alert with AlertManager
Prometheus monitor on data. We can configure some rules apply to data to alert if it's satisfied. That alert which in turn sent to AlertManager. In there, we can configure some notification channel such as Slack, email, MS Teams, etc to inform end-users.

By default, AlertManager is installed with ClusterIP in its service. Need to change it to LoadBalancer to be able to access from outside.

In prometheus.values.yaml, navigate to alertmanager > service, change to:  

```
loadBalancerIP: ""
loadBalancerSourceRanges: []
servicePort: 80
# nodePort: 30000
sessionAffinity: None
type: LoadBalancer
```

Apply your changes to AKS:
```
helm upgrade my-prometheus prometheus-community/prometheus --set server.service.type=LoadBalancer --set rbac.create=false -f prometheus.values.yaml
```

### Link AlertManager to Prometheus
Copy AlertManager external IP, add it to Prometheus configuration.

In prometheus.values.yaml, go to server > alertmanagers, add:  

```
- scheme: http
  static_configs:
    - targets:
        - push-gateway-external-IP-here
```

### Configure alert rules

In serverFiles > alerting_rules.yml, add snippet below to define a rule:
```
groups:
  - name: telemetry
    rules:
      - alert: ProcessingError
        expr: increase(push_counter{job="your-job-name"}[2m]) > 0
        for: 2m
        labels:
          severity: critical
        annotations:
          description: "{{ $labels.instance }} of job {{ $labels.job }} has error in last 2 minutes."
          summary: "Instance {{ $labels.instance }} error"
```

Then, in alertmanagerFiles > alertmanager.yml, add:

```
global:
  slack_api_url: paste-your-slack-channel-webhook-here

receivers:
  - name: slack-receiver
    slack_configs:
      - channel: "#monitoring-aks-using-prometheus"
        send_resolved: true

route:
  group_by: ["telemetry"]
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 5h
  receiver: slack-receiver
```

That's it. You now can see the alert message in Slack channel if something meets your rule.

*Hint*: if you get any problem about configuration, find the corresponding Helm chart template file to understand how it structured.