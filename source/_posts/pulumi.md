---
title: Provisioning a GKE Cluster Using Pulumi
date: 2023-7-19 9:30:00
tags: Cloud
categories: Work
thumbnail: "https://i.imgur.com/yLRvYVC.jpg"
---

Before you begin, make sure you have the following:

- Google Cloud Platform (GCP) account with appropriate permissions.
- Pulumi CLI installed. You can find installation instructions here.
- Go installed. You can find installation instructions here.
- Make sure to check Pulumi documentations

# Google-Cloud-SDK

![](https://i.imgur.com/KVpYJDh.png)

![](https://i.imgur.com/ZLDXrNR.png)

Then

```bash
gcloud auth application-default login
```

# Pulumi Installation

![](https://i.imgur.com/jNrR6WW.png)

# New Project and stack name

- Create a new directory for your project: mkdir pulumi-gke-cluster
- Change into the project directory: cd pulumi-gke-cluster
- Initialize a new Pulumi project with Go: pulumi new gcp-go
- Select the desired project name and stack name when prompted.

This project name allows you to track your changes via the Pulumi Dashboard

In Pulumi, a stack name represents a unique identifier for a specific instance of your infrastructure. It helps you manage different configurations or variations of your infrastructure within a single Pulumi project.

When you create a new Pulumi stack, you provide a stack name to distinguish it from other stacks. The stack name can be any string that meets the naming requirements of your cloud provider.

![](https://i.imgur.com/hmUc53v.png)

For the first time, you will be promoted to enter your auth token which you can get by making a Pulumi account.

# Commands

## pulumi new &lt;template&gt;

- Description: Initializes a new Pulumi project from a template.
- Usage: pulumi new &lt;template&gt;
- Example: pulumi new gcp-go

## pulumi stack select

- Description: Selects an existing Pulumi stack to work with.
- Usage: pulumi stack select &lt;stack-name&gt;
- Example: pulumi stack select dev-cluster

## pulumi up

- Description: Deploys or updates the resources in the current Pulumi stack.
- Usage: pulumi up
- Example: pulumi up

## pulumi destroy

- Description: Destroys all resources in the current stack.
- Usage: pulumi destroy
- Example: pulumi destroy

## pulumi stack ls

- Description: Lists all the available stacks.
- Usage: pulumi stack ls
- Example: pulumi stack ls

## pulumi config

- Description: Manages configuration values for the current stack.
- Usage:
pulumi config set &lt;key&gt; &lt;value&gt;
pulumi config get &lt;key&gt;
pulumi config rm &lt;key&gt;
- Example:
pulumi config set gcpProject my-project-id
pulumi config get gcpProject
pulumi config rm gcpProject

## pulumi stack export

- Description: Exports the current stack's state to a file.
- Usage: pulumi stack export --file &lt;filename&gt;
- Example: pulumi stack export --file stack-state.json

## pulumi stack import

- Description: Imports a stack's state from a file.
- Usage: pulumi stack import --file &lt;filename&gt;
- Example: pulumi stack import --file stack-state.json

# State Management

Pulumi state represents the current state of your infrastructure as managed by Pulumi. It includes resource information, configuration settings, and other relevant metadata.

Pulumi state is stored in a backend, which can be a local file, a cloud storage bucket (our case), or a remote state management service (e.g., Pulumi service, AWS S3, Azure Blob Storage, etc.).

# Provisioning the Cluster

![](https://i.imgur.com/7PrrjKe.png)

Voila! Our cluster

![](https://i.imgur.com/onYn6lx.png)

The NodePools

![](https://i.imgur.com/4mFCcHo.png)

And the bucket

![](https://i.imgur.com/ryHQFpL.png)

Pulumi Dashboard

![](https://i.imgur.com/kTWNdDw.png)

Pulumi Dashboard helps you visualize your infrastructure and track changes over time. It also provides a convenient way to share your infrastructure with your team.

# Code

**Main.go**

```go
package main

import (
	"github.com/pulumi/pulumi-gcp/sdk/v6/go/gcp/storage"
	"github.com/pulumi/pulumi-gcp/sdk/v6/go/gcp/container"
	"github.com/pulumi/pulumi-kubernetes/sdk/v3/go/kubernetes"
	appsv1 "github.com/pulumi/pulumi-kubernetes/sdk/v3/go/kubernetes/apps/v1"
	corev1 "github.com/pulumi/pulumi-kubernetes/sdk/v3/go/kubernetes/core/v1"
	metav1 "github.com/pulumi/pulumi-kubernetes/sdk/v3/go/kubernetes/meta/v1"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)


func generateKubeconfig(clusterEndpoint pulumi.StringOutput, clusterName pulumi.StringOutput,
	clusterMasterAuth container.ClusterMasterAuthOutput) pulumi.StringOutput {
	context := pulumi.Sprintf("demo_%s", clusterName)

	return pulumi.Sprintf(`apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: %s
    server: https://%s
  name: %s
contexts:
- context:
    cluster: %s
    user: %s
  name: %s
current-context: %s
kind: Config
preferences: {}
users:
- name: %s
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: gke-gcloud-auth-plugin
      installHint: Install gke-gcloud-auth-plugin for use with kubectl by following
        https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke
      provideClusterInfo: true
`,
		clusterMasterAuth.ClusterCaCertificate().Elem(),
		clusterEndpoint, context, context, context, context, context, context)
}

func main() {
	pulumi.Run(func(ctx *pulumi.Context) error {
		// Create a GCP resource (Storage Bucket)
		bucket, err := storage.NewBucket(ctx, "my-bucket", &storage.BucketArgs{
			Location: pulumi.String("US"),
		})
		if err != nil {
			return err
		}

		projectName := pulumi.String(ctx.Project())
		clusterName := pulumi.String("my-cluster")
		nodePoolName := pulumi.String("my-node-pool")
		zone := pulumi.String("us-central1-a")

		cluster, err := container.NewCluster(ctx, "gkeCluster", &container.ClusterArgs{
			Project:     projectName,
			Name:        clusterName,
			InitialNodeCount: pulumi.Int(1),
			Location:          zone,
		})
		if err != nil {
			return err
		}

		nodePool, erro := container.NewNodePool(ctx, "gkeNodePool", &container.NodePoolArgs{
			Project:     projectName,
			Cluster:     cluster.Name,
			Name:        nodePoolName,
			InitialNodeCount: pulumi.Int(1),
			Location:          zone,
		}, pulumi.DependsOn([]pulumi.Resource{cluster}))
		if erro != nil {
			return erro
		}

		// Export KubeConfig
		ctx.Export("kubeconfig", generateKubeconfig(cluster.Endpoint, cluster.Name, cluster.MasterAuth))


		k8sProvider, err := kubernetes.NewProvider(ctx, "k8sprovider", &kubernetes.ProviderArgs{
			Kubeconfig: generateKubeconfig(cluster.Endpoint, cluster.Name, cluster.MasterAuth),
		}, pulumi.DependsOn([]pulumi.Resource{cluster}))
		if err != nil {
			return err
		}

		// create deployment and service for nginx
		appLabels := pulumi.StringMap{
			"app": pulumi.String("nginx"),
		}
		deployment, er := appsv1.NewDeployment(ctx, "nginx", &appsv1.DeploymentArgs{
			Metadata: &metav1.ObjectMetaArgs{
				Name:      pulumi.String("nginx"),
				Namespace: pulumi.String("default"),
				Labels:    appLabels,
			},
			Spec: &appsv1.DeploymentSpecArgs{
				Replicas: pulumi.Int(1),
				Selector: &metav1.LabelSelectorArgs{
					MatchLabels: appLabels,
				},
				Template: &corev1.PodTemplateSpecArgs{
					Metadata: &metav1.ObjectMetaArgs{
						Labels: appLabels,
					},
					Spec: &corev1.PodSpecArgs{
						Containers: corev1.ContainerArray{
							corev1.ContainerArgs{
								Name:  pulumi.String("nginx"),
								Image: pulumi.String("nginx"),
								Ports: corev1.ContainerPortArray{
									corev1.ContainerPortArgs{
										ContainerPort: pulumi.Int(80),
									},
								},
							},
						},
					},
				},
			},
		}, pulumi.DependsOn([]pulumi.Resource{nodePool}), pulumi.Provider(k8sProvider))
		if er != nil {
			return er
		}

		_, err = corev1.NewService(ctx, "nginx", &corev1.ServiceArgs{
			Metadata: &metav1.ObjectMetaArgs{
				Name:      pulumi.String("nginx"),
				Namespace: pulumi.String("default"),
				Labels:    appLabels,
			},
			Spec: &corev1.ServiceSpecArgs{
				Type: pulumi.String("LoadBalancer"),
				Ports: corev1.ServicePortArray{
					corev1.ServicePortArgs{
						Port:       pulumi.Int(80),
						TargetPort: pulumi.Int(80),
					},
				},
				Selector: appLabels,
			},
		}, pulumi.DependsOn([]pulumi.Resource{nodePool, deployment}), pulumi.Provider(k8sProvider))
		if err != nil {
			return err
		}

		// Export the DNS name of the bucket
		ctx.Export("bucketName", bucket.Url)

		return nil
	})
}
```

**Pulumi.yaml**
```yaml
name: pcp8-390717
runtime: go
description: A minimal Google Cloud Go Pulumi program
```

**Pulumi.dev.yaml**
```yaml
config:
  gcp:project: pcp8-390717
```