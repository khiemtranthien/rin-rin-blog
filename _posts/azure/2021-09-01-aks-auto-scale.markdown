---
layout: post
title:  "AKS Horizontal Pod Autoscaler"
date:   2021-09-01 20:13:37 +0700
categories: azure-cloud
tags: aks hpa
---

## Architecture

![View orchid](/rin-rin-blog/assets/images/azure/aks/hpa.png)

## Pod Replica Count

For many applications with usage that varies over time, you may want to add or remove pod replicas in response to changes in demand for those applications. The HPA  can manage scaling these workloads for you automatically.

### USE CASES

The HPA is ideal for scaling stateless applications, although it can also support scaling stateful sets. Using HPA in combination with cluster autoscaling (see below) can help you achieve cost savings for workloads that see regular changes in demand by reducing the number of active nodes as the number of pods decreases.

### HOW IT WORKS

For workloads with HPA configured, the HPA controller monitors the workload’s pods to determine if it needs to change the number of pod replicas. In most cases, where the controller takes the mean of a per-pod metric value, it calculates whether adding or removing replicas would move the current value closer to the target value. For example, a deployment might have a target CPU utilization of 50%. If five pods are currently running and the mean CPU utilization is 75%, the controller will add 3 replicas to move the pod average closer to 50%.

HPA scaling calculations can also use custom or external metrics. Custom metrics target a marker of pod usage other than CPU usage, such as network traffic, memory, or a value relating to the pod’s application. External metrics measure values that do not correlate to a pod. For example, an external metric could track the number of pending tasks in a queue.

## Installation

In Helm scripts:

values.yaml, add new property

```
resources:
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
   enabled: true
   minReplicas: 8
   maxReplicas: 20
   targetCPUUtilizationPercentage: 50
   targetMemoryUtilizationPercentage: 50
```

Add new file: hpa.yaml

```
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: {{  .Values.deploymentName   }}
  labels:
     app: {{ .Values.deploymentName }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Values.deploymentName  }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        targetAverageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  {{- end }}
  {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        targetAverageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
  {{- end }}
{{- end }}
```

In pod deployment file, add if statment to check autoscaling and resources for pod

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.deploymentName }}
  labels:
    app: {{ .Values.deploymentName }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
spec:
{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
{{- end }}
  selector:
    matchLabels:
      app: {{ .Values.deploymentName }}
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Values.deploymentName }}
        release: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}"
          volumeMounts:
          - name: {{ .Values.volumeMounts.name }}
            mountPath: {{ .Values.volumeMounts.mountpath }}
            readOnly: true
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
          - name: k8sname
            value: {{ .Values.clusterName }}
          {{- if .Values.env.values }}
          {{- range .Values.env.values }}
          - name: {{ .name }}
            value: {{ .value | quote }}
          {{- end -}}
          {{- end -}}
          {{- if .Values.env.configmap }}
          {{- range .Values.env.configmap }}
          - name: {{ .name }}
            valueFrom:
              configMapKeyRef:
                name: {{ .cfgName }}
                key: {{ .key }}
          {{- end -}}
          {{- end }}
          ports:
            - name: http
              containerPort: {{ .Values.container.ContainerPort }}
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      volumes:
         - name: {{ .Values.Volumes.name }}
           csi:
               driver: {{ .Values.Volumes.driver }}
               readOnly: true
               volumeAttributes:
                  secretProviderClass: {{ .Values.SecretProviderClass.providerClassName }}
               nodePublishSecretRef:
                     name: akv-creds-stores
```

