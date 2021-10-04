---
layout: post
title:  "Install Prometheus on AKS"
date:   2021-10-04 15:13:37 +0700
categories: azure-cloud
tags: prometheus aks
---

## Connect to your AKS cluster

```
AKS_NAME="exp-khiem-aks-cluster"
AKS_RG="exp-khiem-rg-aks-cluster"
AKS_REGION="eastus"
AKS_SUBSCRIPTION="Subscription1"

FILE="$HOME/$AKS_NAME.kubeconfig"

export KUBECONFIG=$FILE

kubectl cluster-info
```

## Install Prometheus with Helm

```
kubectl config set-context --current --namespace=$NAMESPACE

helm repo add stable https://charts.helm.sh/stable

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics

helm repo update

helm install my-prometheus prometheus-community/prometheus --set server.service.type=LoadBalancer --set rbac.create=false

# Verify the installation by looking at your services
kubectl get services
```

## Add metrics to your app

Create simple API .NET core app.

Install Prometheus libraies:
```
Install-Package prometheus-net
Install-Package prometheus-net.AspNetCore
```
Expose a default set of metrics: `app.UseMetricServer();`  
Expose HTTP request metrics: `app.UseHttpMetrics();`  

Startup.cs looks like:  
```
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseRouting();

    app.UseAuthorization();

    app.UseMetricServer();

    app.UseHttpMetrics();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```
Like this HTTP Metrics, we can even export different Health Status Metrics, Network Health Status Metrics, .Net Core Runtime Metrics. But for simplicity's purpose, I am skipping it. But if you are interested you can go through these Nuget package documentation to have a clear understanding.
```
prometheus-net.AspNetCore.HealthChecks
prometheus-net.AspNetCore.HealthChecks
AspNetCore.HealthChecks.SqlServer
AspNetCore.HealthChecks.Network
prometheus-net.SystemMetrics
```

We can even export some Custom Metrics from our application.
```
using Prometheus;
namespace PrometheusMetrics.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class WeatherForecastController : ControllerBase
    {
        Counter counter = Metrics.CreateCounter("my_counter", "weather request count");
        [HttpGet]
        public IEnumerable<WeatherForecast> Get()
        {
            counter.Inc();
            // your code here
        }
    }
}
```

Start your app. Access: http://localhost:port/metrics, you will see `my_counter` metric there.

## Integrate with Prometheus server

Prometheus server works with pull model. Means it will call your app at endpoint /metrics to get data data it needs.  
Step 1: Dockerize your app and deploy to ASK
Step 2: Configure Prometheus targets to connect to your app

### Step 1: Dockerize your app
```
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1-buster-slim AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/core/sdk:3.1-buster AS build
WORKDIR /src
COPY ["./PrometheusMetrics.csproj", "PrometheusMetrics/"]
RUN dotnet restore "PrometheusMetrics/PrometheusMetrics.csproj"
COPY [".", "PrometheusMetrics"]
WORKDIR "/src/PrometheusMetrics"
RUN dotnet build "PrometheusMetrics.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "PrometheusMetrics.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "PrometheusMetrics.dll"]
```

Deploy with Helm:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-prometheus
  labels:
    app: web-prometheus
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-prometheus
  template:
    metadata:
      labels:
        app: web-prometheus
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "80"
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: web-prometheus
          image: expkhiemacr.azurecr.io/web-prometheus:1.0
          ports:
            - name: http
              containerPort: 80
              protocol: TCP

---
apiVersion: v1
kind: Service
metadata:
  name: web-prometheus
spec:
  selector:
    app: web-prometheus
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

```

### Step 2: Configure targets in Prometheus
Make sure your app metrics is available after deployment. Create a service type Load Balancer. Get external IP and check the metrics: http://external-IP/metrics

Open Prometheus configuration file:
```
kubectl edit cm prometheus-server  
```

And add this new job (in scrape_configs section):
```
- job_name: web-prometheus
    static_configs:
    - targets:
    - web-prometheus
```
![View orchid](/rin-rin-blog/assets/images/azure/prometheus/prometheus-targets.png)
