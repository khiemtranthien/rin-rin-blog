---
layout: post
title:  "Create AKS Ingress Controller with Application Gateway"
date:   2021-08-20 20:13:37 +0700
categories: azure-cloud
tags: aks k9s
---

## Overall architecture
![Application Gateway Ingress Controller](/rin-rin-blog/assets/images/azure/aks/agic.png)

## Create Public IP & VNet & Application Gateway

```
az network public-ip create -n myPublicIp -g MyResourceGroup --allocation-method Static --sku Standard
az network vnet create -n myVnet -g myResourceGroup --address-prefix 11.0.0.0/8 --subnet-name mySubnet --subnet-prefix 11.1.0.0/16 
az network application-gateway create -n myApplicationGateway -l canadacentral -g myResourceGroup --sku Standard_v2 --public-ip-address myPublicIp --vnet-name myVnet --subnet mySubnet
```

## Enable Application Gateway Ingress Controller via CLI

```
appgwId=$(az network application-gateway show -n myApplicationGateway -g myResourceGroup -o tsv --query "id") 
az aks enable-addons -n myCluster -g myResourceGroup -a ingress-appgw --appgw-id $appgwId
```

## Test AGIC

### Create the first app

Create a simple API app with .NET Core 3.1. The app has 2 endpoints: /product and /healthz

Yaml file:

```
apiVersion: v1
kind: Pod
metadata:
  name: aks-sample-product
  labels:
    app: aks-sample-product
spec:
  containers:
    - image: "expkhiemacr.azurecr.io/aks_sample_product:1.2"
      name: aks-sample-product
      ports:
        - containerPort: 80
          protocol: TCP

---
apiVersion: v1
kind: Service
metadata:
  name: aks-sample-product
spec:
  selector:
    app: aks-sample-product
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

```

### Create the second app

Create a simple API app with .NET Core 3.1. The app has 2 endpoints: /store and /healthz

Yaml file:

```
apiVersion: v1
kind: Pod
metadata:
  name: aks-sample-store
  labels:
    app: aks-sample-store
spec:
  containers:
    - image: "expkhiemacr.azurecr.io/aks_sample_store:1.2"
      name: aks-sample-store
      ports:
        - containerPort: 80
          protocol: TCP

---
apiVersion: v1
kind: Service
metadata:
  name: aks-sample-store
spec:
  selector:
    app: aks-sample-store
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

```
### Create Ingress

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: aks-sample-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    ingress.kubernetes.io/ssl-redirect: "false"
    appgw.ingress.kubernetes.io/ssl-redirect: "false"
    appgw.ingress.kubernetes.io/health-probe-path: "/healthz"
spec:
  rules:
    - http:
        paths:
          - path: /product
            pathType: Prefix
            backend:
              serviceName: aks-sample-product
              servicePort: 80

          - path: /store
            pathType: Prefix
            backend:
              serviceName: aks-sample-store
              servicePort: 80

```

Then apply it:  

```
kubectl apply -f aks-sample-ingress.yaml

kubectl get ingress

```

Copy Public IP from Ingress and run it in your browser: **http://ingress-IP/product**, **http://ingress-IP>/store**

See [Application Gateway Ingress Controller](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/)
and [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

Note:  
- If you want to match subpath, use wildcard, for example: /store/* will match /store/info
- In case your store service does not have "/store" prefix in API (for example: /api/showrowm), but in application gateway you still want to use "/store/*" to direct URL to store service, use: appgw.ingress.kubernetes.io/backend-path-prefix: "/" in annotation section, it will cut off "/store" in the orginal URL.

## Clean up resources

```
kubectl delete -f aspnetapp.yaml

az group delete --name myResourceGroup 

```