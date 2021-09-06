---
layout: post
title:  "Create AKS cluster with Azure CNI"
date:   2021-08-19 21:13:37 +0700
categories: azure-cloud
tags: aks
---
## Set default active subscription
``` az account list --output table ``` 
``` az account set --subscription "<subscription name>" ```

## Import public Docker image to ACR

```
az acr import  -n PASTE-YOUR-ACR-NAME-HERE --source docker.io/library/nginx:1.19.0 --image nginx:1.19.0
```

## Create an UID

```
RESOURCES_UID=$(uuidgen | cut -c1-8)
echo "RESOURCES_UID: $RESOURCES_UID" 
```

## AKS Network

```
VNET_NAME="vnet-aks-$RESOURCES_UID"
VNET_RG="rg-vnet-aks-$RESOURCES_UID"
VNET_REGION="eastus"
VNET_SUBSCRIPTION="DEV"
```

### Create the VNET RG

```
az group create \
  --name $VNET_RG \
  --location $VNET_REGION \
  --subscription $VNET_SUBSCRIPTION
```

### Create the VNET and Subnet

```
az network vnet create \
  --name $VNET_NAME \
  --resource-group $VNET_RG \
  --address-prefixes 192.0.0.0/8 \
  --subnet-name subnet-aks-$RESOURCES_UID \
  --subnet-prefixes 192.10.0.0/16 \
  --location $VNET_REGION \
  --subscription $VNET_SUBSCRIPTION
```

## Create AKS cluster

### Get subnet ID

```
VNET_NAME="vnet-aks-$RESOURCES_UID"
VNET_RG="rg-vnet-aks-$RESOURCES_UID"
SUBNET_NAME="subnet-aks-$RESOURCES_UID"
VNET_ID=$(az network vnet show --resource-group $VNET_RG --name $VNET_NAME --query id -o tsv)
SUBNET_ID=$(az network vnet subnet show --resource-group $VNET_RG --vnet-name $VNET_NAME --name $SUBNET_NAME --query id -o tsv)
```

### AKS env vars

```
AKS_NAME="aks-cluster-$RESOURCES_UID"
AKS_RG="rg-aks-cluster-$RESOURCES_UID"
AKS_REGION="eastus"
AKS_SUBSCRIPTION="DEV"
```

### AKS RG

```
az group create \
  --location $AKS_REGION \
  --name $AKS_RG \
  --subscription $AKS_SUBSCRIPTION
```

### Create SP

```
az ad sp create-for-rbac \
  --skip-assignment \
  -n "sp-aks"
```

### Add AKS SP Owner of the VNET RG

```
az role assignment create \
  --role "Owner" \
  --assignee "..." \
  --resource-group $VNET_RG
```
- Wait 15 minutes after assigning the role, permission propagation can be crazy sometimes. 
- In my case, I don't have permission to run above command so I ask the support from another account that has Admin Access or Owner permission.

### Login with new SP that has Owner role

- Open another terminal, login with SP that has assigned Owner role above

```az login --service-principal -u <app-id> -p <password-or-cert> --tenant <tenant>```

### Update SP credential to AKS if you reset password

Some action on AKS may be failed because of in valid SP secret token because you have reset the password before, let update it:  

```az aks update-credentials --resource-group $AKS_RG --name $AKS_NAME --reset-service-principal --service-principal $SP --client-secret $SP_SECRET```


### Create AKS

```
az aks create \                     
  --location $AKS_REGION \
  --subscription $AKS_SUBSCRIPTION \
  --resource-group $AKS_RG \
  --name $AKS_NAME \
  --ssh-key-value $HOME/.ssh/id_rsa.pub \
  --service-principal "PASTE-YOUR-SP-ID-HERE" \
  --client-secret "PASTE-YOUR-SP-PASSWORD-HERE" \
  --network-plugin kubenet \
  --load-balancer-sku standard \
  --outbound-type loadBalancer \
  --vnet-subnet-id $SUBNET_ID \
  --pod-cidr 10.244.0.0/16 \
  --service-cidr 10.0.0.0/16 \
  --dns-service-ip 10.0.0.10 \
  --docker-bridge-address 172.17.0.1/16 \
  --node-vm-size Standard_B2s \
  --enable-cluster-autoscaler \
  --max-count 5 \
  --min-count 2 \
  --node-count 2 \
  --attach-acr "PASTE-YOUR-ARC-NAME-HERE" \
  --tags 'ENV=DEV' 'SRV=EXAMPLE'
```
- If your ACR in different resource group, get and specify ACR_ID to --attach-acr  
```ACR_ID=$(az acr show -n <acr name> -g <resource group> --query id -o tsv)```

- Can ignore attach ACR when creating AKS and attach it later:  
```az aks update -n $AKS_NAME -g $AKS_RG --attach-acr $ACR_NAME ```

Get kubeconfig.

```
FILE="./$AKS_NAME.kubeconfig"

az aks get-credentials \
  -n $AKS_NAME \
  -g $AKS_RG \
  --subscription $AKS_SUBSCRIPTION \
  --admin \
  --file $FILE
  
export KUBECONFIG=$FILE

kubectl cluster-info
```

## Deploy an app

Create a namifest file.

```
vim my-app.yaml
```

With the following content.

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: PASTE-YOUR-ACR-SERVER-HERE/IMAGE-REPO:TAG
        name: my-app
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: my-app
  type: LoadBalancer
```

Apply the manifest.

```
kubectl apply -f my-app.yaml
```

## K8s Dashboard

Go to: https://github.com/kubernetes/dashboard

And get the latest version.

### Dashboard permissions

You have two options:

1. Use your kubeconfig file;
2. Create a SA and use its token.

### RBAC

```
cat > ./dashboard-rbac.yml <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin-user
  namespace: kubernetes-dashboard

EOF
```

Apply the manifest.

```
kubectl apply -f ./dashboard-rbac.yml
```

### Get the token

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secrets |grep dashboard-admin-user-token | awk '{print $1}')
```

### Access the dashboard

```
kubectl proxy
```

Access: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

### Expose the dashboard to the Internet (please don't)

Create a manifest.

```
vim expose.yaml
```

With the following content

```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-internet
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  type: LoadBalancer
```

Apply the manifest.

```
kubectl apply -f expose.yaml -n kubernetes-dashboard
```

### Login K8s Dashboard with Kubeconfig file

Choose Kubeconfig file option in Dashboard login page, browse to your kubeconfig file in your home folder and go. That's it.

## Clean up AKS resources

```az group delete --name $AKS_RG --yes --no-wait```