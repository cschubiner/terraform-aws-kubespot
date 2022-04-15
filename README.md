# opsZero Kubespot (AWS)

<img src="http://assets.opszero.com.s3.amazonaws.com/images/auditkube.png" width="200px" />

Compliance Oriented Kubernetes Setup for AWS, Google Cloud and Microsoft Azure.

Kubespot is an open source terraform module that attempts to create a complete
compliance-oriented Kubernetes setup on AWS, Google Cloud and Azure.  These add
additional security such as additional system logs, file system monitoring, hard
disk encryption and access control. Further, we setup the managed Redis and SQL
on each of the Cloud providers with limited access to the Kubernetes cluster so
things are further locked down. All of this should lead to setting up a HIPAA /
PCI / SOC2 being made straightforward and repeatable.

This covers how we setup your infrastructure on AWS, Google Cloud and Azure.
These are the three Cloud Providers that we currently support to run Kubernetes.
Further, we use the managed service provided by each of the Cloud Providers.
This document covers everything related to how infrastructure is setup within
each Cloud, how we create an isolated environment for Compliance and the
commonalities between them.

# Tools & Setup

```
brew install kubectl kubernetes-helm awscli google-cloud-sdk azure-cli terraform packer
```

# Credentials

## AWS

Add your IAM credentials in ~/.aws/credentials.

```
[profile_name]
aws_access_key_id=<>key>
aws_secret_access_key=<secret_key>
region=us-west-2
```

```
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com
```

# Network Diagram

We setup Kubernetes using the managed service provider on each of the Cloud
providers. AWS EKS, Google Cloud GCE, Azure AKS. This ensures that we don’t need
to handle running the master nodes which can create additional operational
hurdles. We remove this from the picture as much as possible.

Kubernetes will be running with the following things:

 - Ingress controller to reduce the expense of running multiple LoadBalancers
Nodes
Nodes can be configured using Terraform. Each of the modules for EKS, GCP, AKS
have configuration options for adding and additional nodes. Further, you can
specify the size and type of the nodes using the Terraform script as well. This
should be the variables min_size and max_size. The amount of nodes that a master
does not need to be configured and is handled by the managed service providers.

The way to add additional nodes to the cluster is to increase the min_size of
the nodes. This will create additional nodes in the cluster. Note that it may
take up to 5 minutes to bring up additional nodes but there is not downtime. You
can also do the same by reducing the min_size. This will remove the pods and
move them to different nodes. Ensure that your code is idempotent to handle
cases where the service may be killed.

The way to increase the size is to modify the terraform script and run terraform
apply. This will update the configuration. Further, with Azure and GCP we can
specify auto update. With EKS there needs to be a manual process for building
the nodes updating and replacing them which is described in the AWS section.

Request Cycle

Pods are a group of containers. They are in the simplest form a group of pods
that run on the same node together. The way you specify the pods is through a
deployment and how you expose these to the outside world is through a service
and ingress. An example of a HTTP request looks like this:

```
DNS (i.e app.example.com) -> Ingress (Public IP Address/CNAME) -> Kubernetes Service -> Kubernetes Pods
```


Monitoring

Monitoring is configured through third party services such as Datadog, New
Relic, etc. These services will cover what the issues with the pods are and
other metrics. The need to be setup separately but all of them provide a Helm
chart to install so no additional configuration is needed.


# Terraform

 - [AWS EKS](examples/eks)
 - [Google Cloud](examples/gcp)
 - Microsoft Azure

The infrastructure is setup using Kubespot which is a Terraform module to
create the entire infrastructure. Terraform is used to create Infrastructure as
Code so you don’t have to go into the Consoles of the different environments and
point and click to build infrastructure.

opsZero setups the infrastructure using Terraform so that it can be built in a
repeatable manner. This grants you a couple benefits: it creates an audit trail
of changes to your infrastructure so you remain compliant, it allows you test
new infrastructure services quickly if you want to add them, it allows you to
create completely identical isolated environment across different Cloud
environments.

Our Terraform module creates the following across different modules: Kubernetes
Cluster, Bastion, VPN Machine, SQL (AWS Aurora, AWS RDS, Google Cloud SQL, Azure
Database for PostgreSQL), and Redis (AWS ElasticCache, Google MemoryStore, Azure
Redis), VPCs, Security Groups.

We setup a new Virtual Private Clouds (VPCs) that isolate the access in each
environment. This is beneficial in that even if you are using an existing Cloud
environment the VPC in which Kubernetes is deployed will be isolated from the
other networks unless it is opened up via VPC Peering. Also by having everything
within one VPC we can create and limit network flows to the required services.

Since Terraform is just code it allows us to check in all changes into Git to
create an audit trail. This audit trail and all changes to the infrastructure
need to be documented to remain compliant with HIPAA / PCI / SOC2.

The Bastion and VPN are two separate machines that have an external IP. These
are how we connect to the Kubernetes cluster as it requires we connect to the
VPN and then to the Bastion to have access to the Kubernetes cluster. We use
Foxpass for authentication to the Bastion and VPN. Foxpass allows you to use G
Suite and Office 365 to grant access to the machines giving a singular place for
access.

Terraform needs only be run when we create the infrastructure and when we want
to make changes to that infrastructure. The way terraform works is that it
creates the infrastructure and generates a statefile when you run terraform
apply. This file is the state of your infrastructure and should be checked in to
Git. Additional runs of terraform apply compares this statefile to what exists
in your infrastructure and creates, modifies or deletes based on what is in your
terraform .tf file and what your statefile shows.

The usual reasons you would run terraform are:

Change the number of nodes running in your cluster
Change the size of your database
Change the size of your redis
Add additional services to your infrastructure
AWS
The configuration for AWS looks something like this:

We build a completely independent VPC that is locked down. We lock things down by doing the following:

Need to use bastion for access. It uses Foxpass for access through G Suite, Microsoft 360, OKTA.
Need to use VPN for access to the bastion.
Need to use ELB via Ingress to Access Kubernetes Services
Additional Logging and Security Updates on Amazon Linux including OSSEC
Additional Control Log Flows
Node level Encryption
Google Cloud
Azure


# Preview Environnments

 - Github Actions

# Kubernetes

## kubeconfig

### AWS

Ensure you have access to EKS. This is done by adding your IAM to the EKS in the
terraform configuration.

Get the credentials

```
KUBECONFIG=./kubeconfig aws --profile=account eks update-kubeconfig --cluster cluster-name
```

There can be multiple clusters so pick the correct cluster.

Ensure that you set `export KUBECONFIG=./kubeconfig` to get the correct `KUBECONFIG`
file. This can be added into you .bashrc or .zshrc

## Pods
### List Running Pods

``` sh
kubectl get pods -A
```

Kubernetes lets you divide your cluster into namespaces. Each namespace can have
its own set of resources. The above command lists all running pods on every
cluster. Pods in the kube-system namespace belong to Kubernetes and helps it
function.


### “SSH”

To connect to the application look at the namespaces:

``` sh
kubectl get pods --all-namespaces
kubectl exec -it -n <namespace> <pod> -c <container> -- bash
```


### Logs

``` sh
kubectl get pods --all-namespaces
kubectl logs -f -n <namespace> <pod> -c <container>
```

This lets you view the logs of the running pod. The container running on the pod
should be configured to output logs to STDOUT/STDERR.

### Describe Pods

Troubleshooting Pods

kubectl describe pods

Common Errors:

 - OOMError
 - CrashLoopBackup
 - ImageNotFound

## Restart a Pod

If you pod is not responding or needs a restart the way to do it is to use the
following command. This will delete the pod and replace it with a new pod if it
is a part of a deployment.

kubectl delete pod <pod-name>

## Deployments

### Scale Down / Up

This has to be done through the deployment in the helm chart. Another way to do it is to scale down

```
num=0

kubectl scale --replicas=${num} -n <namespace> deployment/<deploymentname>
```

## Nodes

Restarting nodes may need to happen if you need to change the size of the
instance, the machine's disk gets full, or you need to update a new AMI.  The
following code provides the howto.

### EKS


```
export AWS_PROFILE=aws_profile
export KUBECONFIG=./kubeconfig

for i in $(kubectl get nodes | awk '{print $1}' | grep -v NAME)
do
        kubectl drain --ignore-daemonsets --grace-period=60 --timeout=30s --force $i
        aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances --filter "Name=private-dns-name,Values=${i}" | jq -r '.Reservations[].Instances[].InstanceId')
        sleep 300 # Wait 5 mins for the new machine to come back up
done
```



# Helm

## Ingress

The ingress is in its simplest form a Kubernetes LoadBalancer. Instead of what
would traditionally be this:

``` sh
DNS (i.e app.example.com) -> Kubernetes Service -> Kubernetes Pods
```

It is the following

``` sh
DNS (i.e app.example.com) -> Ingress (Public IP Address/CNAME) -> Kubernetes Service -> Kubernetes Pods
```

To break down the Ingress request cycle even further it is the following:

``` sh
DNS (i.e app.example.com) -> Ingress [Kubernetes Service -> Kubernetes Pods (Nginx) -> Kubernetes Service -> Kubernetes Pods]
```

The Ingress is just another pod such as Nginx that relays the traffic and as
such is just another pod in the system. The ingress is a helm chart and is
installed manually with the following script.

The ingress works at the DNS layer so it needs to be passed a Host to work:

``` sh
curl -k -H "Host: app.example.com" <https://a54313f35cb5b11e98bb60231b063008-2077563408.us-west-2.elb.amazonaws.com>
```

## Releases

```sh
TAG=v3.0.1
gh release create $TAG --discussion-category "General"
```

# Support
<a href="https://www.opszero.com"><img src="http://assets.opszero.com.s3.amazonaws.com/images/opszero_11_29_2016.png" width="300px"/></a>

This project is by [opsZero](https://www.opszero.com). We help organizations
migrate to Kubernetes so [reach out](https://www.opszero.com/#contact) if you
need help!

# License

This Source Code Form is subject to the terms of the Mozilla Public
License, v. 2.0. If a copy of the MPL was not distributed with this
file, You can obtain one at http://mozilla.org/MPL/2.0/.
