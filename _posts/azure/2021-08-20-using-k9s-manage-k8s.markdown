---
layout: post
title:  "Using K9s to manage AKS"
date:   2021-08-20 17:13:37 +0700
categories: azure-cloud
tags: aks k9s
---

[Link to github page](https://github.com/derailed/k9s)

## Install in MacOS

`brew install k9s`

## Run in command line

`k9s`

![View orchid](/rin-rin-blog/assets/images/azure/aks/k9s.png)

## See pods

Type ``` : pod ```

Access to running pod: `kubectl exec --stdin --tty shell-demo -- /bin/bash`

## See services

Type ``` : service ```

## Exit

``` Ctrl + C ```