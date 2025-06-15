# VernissageK8s

## Overview

The manifests in `manifests/` were adapted from the official Vernissage documentation for a Docker deployment: [https://docs.joinvernissage.org/documentation/vernissageserver/dockercontainers](https://docs.joinvernissage.org/documentation/vernissageserver/dockercontainers).

The docker-compose file is stored in `source_files/` for reference.

This is not a Helm chart. The manifests in `manifests/` are intended to be applied directly via `kubectl apply -f MANIFEST_NAME.yaml`.

## Prerequisites

### CoreDNS Corefile update

Find the ConfigMap in your cluster that contains the CoreDNS Corefile. This is usually named `coredns` in the `kube-system` namespace.

You will need to add the following lines to the Corefile to allow Vernissage to resolve its own internal services:

```
rewrite name vernissage-api.internal vernissage-api.vernissage.svc.cluster.local.
rewrite name vernissage-web.internal vernissage-web.vernissage.svc.cluster.local.
rewrite name vernissage-proxy.internal vernissage-proxy.vernissage.svc.cluster.local.
rewrite name vernissage-push.internal vernissage-push.vernissage.svc.cluster.local.
rewrite name vernissage-redis.internal vernissage-redis.vernissage.svc.cluster.local.
```

I have included my Corefile in this repository at [Corefile](Corefile), to make it easier to know where to add these lines.

Edit the Corefile ConfigMap in your cluster with the following command:

```bash
kubectl edit configmap coredns -n kube-system
```

Add the lines above to the Corefile, save, and exit.

## Order of Operations

1. Namespace
1. ConfigMaps
1. Postgres storage (PersistentVolumeClaim)
1. Postgres database
1. Redis cache
1. Vernissage API backend
1. Vernissage Web frontend
1. Vernissage HTTP proxy
1. Vernissage WebPush server

## Deployment Instructions

### Create the namespace

All the resources in the `manifests/` directory need to live in a namespace.

The resources, as written, use the `vernissage` namespace. You can change this in the manifests if you prefer a different namespace.

To create the namespace, run the following command:

```bash
kubectl create namespace vernissage
```

### Deploy the ConfigMaps

The following manifests in the `manifests/` directory are ConfigMaps:

- [`env.yaml`](manifests/env.yaml)
  - The environment variables listed in this file are sourced from `source_files/env`, from the original Docker deployment. Environment variables that I did not use are prefixed with "z", but are left in `env.yaml` in case they are needed in the future.
- [`postgres-secret.yaml`](manifests/postgres-secret.yaml)
  - Remember to set your Postgres password in this file.

**Look through both files and set values the way that you want them.**

Apply them as follows:

```bash
kubectl apply -f manifests/env.yaml
kubectl apply -f manifests/postgres-secret.yaml
```

### Deploy the Postgres storage volume

These instructions assume that you are familiar with PersistentVolumes and PersistentVolumeClaims.

**The PVC manifest in this rpeository is just an example. I built it with the Rancher UI. It probably will not work if you directly apply it via kubectl. I left the PVC manifest in this repository so that you can see what values may be relevant to set.**

Before deploying the Postgres database, you need to create a PersistentVolumeClaim for the Postgres storage.
The manifest for the PersistentVolumeClaim is in `manifests/postgres-pvc-vernissage.yaml`.

Look through that file and confirm that it will work in your cluster -- for example, you may need to change the storageClassName to a storageClass that you have in your cluster.

Find what storageClassNames you have available in your cluster with the following command:

```bash
kubectl get storageclass
```

Notes about the Postgres PVC that you should set in your version of the manifest:

- `accessModes`: should be ReadWriteMany, in case you have multiple replicas of the Postgres database pods.
- `resources.requests.storage`: should be at least 10Gi, but you can set it to whatever you want. This is the size of the Postgres storage volume.
- `name`: keep it as `postgres-pvc-vernissage` -- if you change it, you will need to modify the `postgres.yaml` Deployment manifest to match the name you set.

Once you are satisfied with your PVC manifest, apply it with the following command:

```bash
kubectl apply -f manifests/postgres-pvc-vernissage.yaml
```

### Deploy the Postgres database

Deploy the Postgres database with the following command:

```bash
kubectl apply -f manifests/postgres.yaml
kubectl apply -f manifests/postgres-service.yaml
```

### Deploy the Redis cache

Deploy the Redis cache with the following command:

```bash
kubectl apply -f manifests/vernissage-redis.yaml
kubectl apply -f manifests/vernissage-redis-service.yaml
```

### Deploy the Vernissage API backend

Deploy the Vernissage API backend with the following command:

```bash
kubectl apply -f manifests/vernissage-api.yaml
kubectl apply -f manifests/vernissage-api-service.yaml
```

### Deploy the Vernissage Web frontend

Deploy the Vernissage Web frontend with the following command:

```bash
kubectl apply -f manifests/vernissage-web.yaml
kubectl apply -f manifests/vernissage-web-service.yaml
```

### Deploy the Vernissage HTTP proxy

Deploy the Vernissage HTTP proxy with the following command:

```bash
kubectl apply -f manifests/vernissage-proxy.yaml
kubectl apply -f manifests/vernissage-proxy-service.yaml
```

### Deploy the Vernissage WebPush server

Deploy the Vernissage WebPush server with the following command:

```bash
kubectl apply -f manifests/vernissage-push.yaml
kubectl apply -f manifests/vernissage-push-service.yaml
```

### Verify the deployment

You can verify that all the pods are running with the following command:

```bash
kubectl get pods -n vernissage
```

You should see all the pods in the `vernissage` namespace in a `Running` state. If any pods are not running, you can check the logs of the pod with the following command:

```bash
kubectl logs POD_NAME -n vernissage
```

If you need to troubleshoot a specific pod, you can also use the following command to get more information about the pod:

```bash
kubectl describe pod POD_NAME -n vernissage
```

## Accessing Vernissage

Once all the pods are running, you can access Vernissage via the HTTP proxy service.

The HTTP proxy service routes traffic to either the Vernissage Web frontend or the Vernissage API backend, depending on the URL path.

The way I accessed my installation of Vernissage was by running `cloudflared` in my cluster, and configuring `cloudflared` to route a public hostname to the following kubernetes-cluster-internal service:

```
http://vernissage-proxy.vernissage.svc.cluster.local.:8080
```

You can also use `kubectl port-forward` to access the Vernissage HTTP Proxy locally:

```bash
kubectl port-forward service/vernissage-proxy -n vernissage 8080:8080
```

Then you can access Vernissage at `http://localhost:8080`.

## Additional Notes

- Make sure to make your S3 bucket public (if on backblaze B2), so that Vernissage can provide URLs to images in it, to web clients, as part of the `VERNISSAGE_CSP_IMG` env var.
  - If you do not want to host a public S3 bucket, you'll need to run a some kind of proxy that can serve up the bucket contents, and use that proxy URL for `VERNISSAGE_CSP_IMG`.
    - An example S3 proxy is the [NGINX S3 Gateway](https://github.com/nginx/nginx-s3-gateway) -- docs are in [the docs folder](https://github.com/nginx/nginx-s3-gateway/blob/main/docs/getting_started.md).
- **How to get WebPush notifications to work:**
  - [These instructions worked for me](https://wiki.pnpde.social/books/vernissage/page/vernissage-hosten-webpush-einrichten)
  - Generate a VAPID key pair using the following site: [VAPID Key Generator](https://www.attheminute.com/vapid-key-generator)
  - WebPush does NOT work in Incognito mode! WebPush requires a regular browser window. I don't understand why yet, but that's how it works.
- Vernissage can only run on a single domain/URL.
