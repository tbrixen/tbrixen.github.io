---
title: "EKS Cluster Games: Challenge 1"
date: 2023-11-09T11:28:47+02:00
draft: false
toc: true
series:
  - EKS-cluster-games
---

This is my writeup of how I solved the first challenge of the [EKS Cluster Games](https://eksclustergames.com/).

## Challenge 1 - Secret Seeker

The intro text told me that the flag is probably among the Kubernetes secrets.

> Jumpstart your quest by listing all the secrets in the cluster. Can you spot the flag among them?

Listing all secrets showed a possible candidate: `log-rotate`

```shell
$ kubectl get secret log-rotate
NAME         TYPE     DATA   AGE
log-rotate   Opaque   1      30h
```

I tried to use use `get` and `describe` to tease out the secret, but no avail

```shell
$ kubectl get secret log-rotate
NAME         TYPE     DATA   AGE
log-rotate   Opaque   1      30h

$ kubectl describe secret log-rotate
Name:         log-rotate
Namespace:    challenge1
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
flag:  52 bytes
```

In the end, I discovered I could use `get` with the `-o json` to get the secret in Base64.

```shell
$ kubectl get secret -o jsonpath='{.items[0].data}'
{"flag":"d2l6__REDACTED__zfQ=="}
```

Base64 decoding it brought out the flag :rocket:

```shell
$ kubectl get secret -o jsonpath='{.items[0].data}' | jq -r '.["flag"]' | base64 --decode
wiz_eks_challenge{REDACTED}%
```
