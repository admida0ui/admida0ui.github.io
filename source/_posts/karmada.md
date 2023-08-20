---
title: Multi-Cloud Multi-Cluster Kubernetes Orchestration with Karmada
date: 2023-8-19 9:30:00
tags: Cloud
categories: Work
thumbnail: "https://i.imgur.com/1xrxNAA.png"
---

## Introduction

Karmada is a multi-Kubernetes cluster application distribution management system that can create multi-cluster applications just like using Kubernetes native API.

## Environment Setup

- We are going to install Karmada kubernetes plugin into our Kind local cluster (on-prem) to work as our host cluster and then it will be joining two member clusters from two different cloud providers Google Cloud Platform and Digital Ocean.
- Both members will join the host in push mode (agentless).

### Prerequisites
Go (Latest)
Kubectl
Kind

### Installation
Create a Kind cluster
```bash
kind create cluster
```

Host Machine (always)
```bash
wget https://github.com/karmada-io/karmada/releases/download/v1.2.0/kubectl-karmada-linux-amd64.tgz

tar -zxf kubectl-karmada-linux-amd64.tgz
sudo cp kubectl-karmada /usr/local/bin/
```

### Initialize the Karmada control plane components on our Kind local cluster
```bash
kubectl karmada init --kubeconfig=~/.kube/config
```

Output:

![](https://i.imgur.com/5yTKO4G.png)

Checking with `kubectl -n karmada-system get pods`

![](https://i.imgur.com/L3GaEPh.png)

### Create Google Cloud GKE Cluster and get its Kubeconfig

![](https://i.imgur.com/ukZgmJ4.png)

Notice, for GKE Kubeconfig we must install the gcloud cli and the gcloud auth plugin for kubectl in order to be able to authenticate to the GKE cluster, this is only required for GKE as Digital Ocean Cluster uses token-based authentication

gcloud requirements installation

```bash
yay -S google-cloud-cli
yay -S google-cloud-cli-gke-gcloud-auth-plugin
```

Add the following to your .bashrc or .zshrc
```bash
export USE_GKE_GCLOUD_AUTH_PLUGIN=True

source ~/.bashrc
```

Getting the GKE Kubeconfig file locally
```
gcloud container clusters get-credentials <clustername> --region us-central1 --project <projectname>
```

Then download the file `~/.kube/config` locally and rename it to member1 or like config1

### Create Digital Ocean Kubernetes Cluster and get its Kubeconfig

![](https://i.imgur.com/nhtq1PJ.png)

Go to actions and download config locally then rename it as well to member2 or config2

With everything done in enviornment setup, we join both clusters to our host cluster now

## Join member clusters

To make names simpler we `sed -i 's/do-ams3-k8s-1-27-4-do-0-ams3-1692406742545/docluster/g' config2` and `sed -i 's/gke_esoteric-might-387308_us-central1_autopilot-cluster-1/gke-cluster/g' config1` because Karmada doesnt support special characters in names as well as longer than 48-character names.

There are two contexts in Karmada:
- karmada-apiserver: `kubectl config use-context karmada-apiserver`
- karmada-host: `kubectl config use-context karmada-host`

The karmada-apiserver is the main kubeconfig to be used when interacting with the Karmada control plane, while the karmada-host is only used for debugging Karmada installation with the host cluster.

Remember, although we have the context karmada-apiserver, this still not an actual Kubernetes cluster, rather it is a Karmada control plane API server running inside the Karmada-host Cluster.

Karmada simplifies multi-cluster access through its central access portals on the karmada-apiserver. This allows unified authentication to be managed within the Karmada control plane. Karmada achieves unified authentication using Kubernetes user camouflage feature.

When a cluster is added to Karmada, it generates a ServiceAccount called karmada-impersonator in the cluster's karmada-cluster namespace. The associated token is then gathered into the Karmada control plane. This ServiceAccount lacks default permissions as it's not bound to any RBAC.

So, let's start with the GKE Cluster

### Join GKE Cluster

**In host cluster, run**
```bash
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config  join gke-cluster --cluster-kubeconfig=~/clusters/config1
```

This will fail saying "no secret was found for the gke-cluster service account", because Kubernetes no longer creates a secret token for service account automatically, so we need to create that manually. Karmada needs to use ServiceAccount and Secret to access member clusters.

So, lets make thing more clear, upon running that command, karmada will create a service account inside our GKE cluster, but the thing is, this service account which is named karmada-gke-cluster lacks a secret, and since Kubernetes decided to no longer dealing with that, we have to deal with it manually...

As shown here:

**Using GKE config to edit service account and create secrets using kubectl**
```bash
$ kubectl --kubeconfig=config2 get sa -n karmada-cluster                                                                
NAME                   SECRETS   AGE
default                0         7h51m
karmada-gke-cluste     0         109m
karmada-impersonator   0         7h51m
```

so let's prepare our secret.yaml file
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: karmada-impersonator
  namespace: karmada-cluster
  annotations:
    kubernetes.io/service-account.name: "karmada-impersonator"
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: Secret
metadata:
  name: gke-cluster-secret
  namespace: karmada-cluster
  annotations:
    kubernetes.io/service-account.name: "karmada-gke-cluster"
type: kubernetes.io/service-account-token
```

and edit the existing service account to append the secret name at the end
![](https://i.imgur.com/s15ZbBE.png)

![](https://i.imgur.com/tlyGNmB.png)

We're done for GKE, let's just join it as member now

This is the output to expect
```bash
$ kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config  join gke-cluster --cluster-kubeconfig=~/clusters/config1
cluster(ubuntub) is joined successfully
```

### Join Digital Ocean Cluster

**In host cluster, run**
```bash
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config  join docluster --cluster-kubeconfig=~/clusters/config2
```

The same process here, Karmada will create a service account for both the docluster and the karmada-impersonator, and both will be lacking secrets.

**Using Digital Ocean config to edit service account and create secrets using kubectl**

![](https://i.imgur.com/CLIAwjq.png)

Secret file here

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: karmada-impersonator
  namespace: karmada-cluster
  annotations:
    kubernetes.io/service-account.name: "karmada-impersonator"
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: Secret
metadata:
  name: docluster-secret
  namespace: karmada-cluster
  annotations:
    kubernetes.io/service-account.name: "karmada-docluster"
type: kubernetes.io/service-account-token
```

### View member clusters

![](https://i.imgur.com/vp0jvTi.png)

## Deploy Nginx application

```bash
$ cat nginx-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
$ kubectl  --kubeconfig /etc/karmada/karmada-apiserver.config apply -f nginx-deployment.yaml
```

Karmada, works on Kubernetes manifests as they are, but requires a distribution strategy and specification of member clusters, so let's create `PropagationPolicy.yaml`

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
    - apiVersion: v1
      kind: Service
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
        - ubuntub
        - ubuntuc
    replicaScheduling:
      replicaDivisionPreference: Weighted
      replicaSchedulingType: Divided
      weightPreference:
        staticWeightList:
          - targetCluster:
              clusterNames:
                - ubuntub
            weight: 1
          - targetCluster:
              clusterNames:
                - ubuntuc
            weight: 1
```

The same could be achieved using a simpler strategy to duplicate across all the member clusters.

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    replicaScheduling:
      replicaSchedulingType: Duplicated
```

So when we apply any of the above PropagationPolicies, both member clusters will have nginx deployment with two replicas.

```bash
kubectl  --kubeconfig /etc/karmada/karmada-apiserver.config apply -f PropagationPolicy.yaml
```

## Application Deployment Status
### In Control Cluster

![](https://i.imgur.com/ttWWFrV.png)

Notice there are no pods here!

### In Member Clusters

![](https://i.imgur.com/NKj50zB.png)

## What to check next?

Implemeting this whole logic in a kubernetes custom controller using Kubebuilder or client-go