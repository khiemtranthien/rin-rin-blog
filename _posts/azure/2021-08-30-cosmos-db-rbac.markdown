---
layout: post
title:  "Grant permission on database level in CosmosDB"
date:   2021-08-30 20:13:37 +0700
categories: azure-cloud
tags: azure cosmos
---

## List of role

```
DB_NAME=hiptechcosmosdb
RG=HiptechRG
az cosmosdb sql role definition list --account-name $DB_NAME --resource-group $RG
```

## Steps

### Step 1:

Create service principal in portal (App Registration menu). It will automatically create a application object in Enterprise application. Add secret for newly created SP in (App Registrations > Cert & Secret menu)

### Step 2:

Get Object ID of SP in **Enterprise applications** menu.

### Step 3:

Assgin role to that SP on the Cosmos DB that want to grant permission.

### Step 4:

Provide SP credential to .NET app to create Cosmos client.