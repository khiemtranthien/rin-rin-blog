---
layout: post
title:  "Push image to Azure Container Registry"
date:   2021-08-23 20:13:37 +0700
categories: azure-cloud
tags: acr
---

## Create ACR

- Create ACR in portal
- Enable admin user on section **Settings > Access keys > Admin user**

## Push image to ACR

### Step 1
```
az login
```

### Step 2
```
az configure --defaults acr=<your acr name>
```

### Step 3
```
az acr build -t <image name>:<tag> .
```

### Step 4

After build, the image is automatically pushed to ACR. Check it in Repositories menu.