---
layout: post
title:  "Secrets Store CSI Driver connect Azure Key Vault to AKS"
date:   2021-08-23 20:13:37 +0700
categories: azure-cloud
tags: key-vault, aks
---

## Architecture

![CSI Architecture](/rin-rin-blog/assets/images/azure/aks/keyvault-csi.png)

## Enable add-on

Microsoft page link [Use the Secrets Store CSI Driver for Kubernetes in an Azure Kubernetes Service (AKS) cluster (preview)](https://docs.microsoft.com/en-us/azure/aks/csi-secrets-store-driver)

```
az feature register --namespace "Microsoft.ContainerService" --name "AKS-AzureKeyVaultSecretsProvider"
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AKS-AzureKeyVaultSecretsProvider')].{Name:name,State:properties.state}"
az provider register --namespace Microsoft.ContainerService
az feature register --namespace Microsoft.ContainerService -n AutoUpgradePreview
```

### Install the aks-preview CLI extension

```
# Install the aks-preview extension
az extension add --name aks-preview

# Update the extension to make sure you have the latest version installed
az extension update --name aks-preview
```

## Enable CSI

```
az aks enable-addons --addons azure-keyvault-secrets-provider --name myAKSCluster --resource-group myResourceGroup
```
### Enabling auto rotation

It is **required for updating the pod mount** and the Kubernetes Secret defined in secretObjects of the SecretProviderClass by polling for changes every two minutes. If not, no secret from SecretProviderClass created. 

``` az aks update -g myResourceGroup -n myAKSCluster2 --enable-secret-rotation ```

## Testing

View this video to better understand how using Key Vault as a secret store in AKS. [Securing Kubernetes Secrets with Azure Key Vault](https://www.youtube.com/watch?v=KGjxZmlb4Bo&t=28s&ab_channel=ZoomSpeaksTech)

[Follow step by step here](https://azure.github.io/secrets-store-csi-driver-provider-azure/demos/standard-walkthrough/) Note: ignore step 1 because already done above.

- Flow:
    1. Make sure you have a SP can access to Key Vault and has permissions to read data from it
    2. Create Kubernetes secret (name secrets-store-creds in this example) to hold SP secret. Later on, it will be used to connect to Key Vault and get out the secret there.
    3. Create SecretProviderClass with SecretObjects (name foosecret in this example)
    4. Create Volume in Deployment refer to SecretProviderClass. Mount volume to pod.
    5. Declare env varible read from kube secret (foosecret).
    6. Deploy your pod, service, ingress, secret provider to AKS, test your app



