---
title: "Clusters: EKS - Migrating Node Groups with Zero Downtime"
menu:
  reference:
    parent: tutorials-kubernetes
    weight: 3
---

In this tutorial we'll launch a new, managed Kubernetes cluster in Elastic
Kubernetes Service (EKS) on AWS. Then, we'll walk through Day-2 operations for
Kubernetes to help you administer and update the cluster, and your workloads running on it.

We'll showcase how to:

1. Create various worker node groups with different settings, such as instance types and AMIs.
1. Deploy a workload to the set of node groups: the [NGINX Ingress Controller][ingress-nginx], and a
simple [echoserver][echoserver] app that echo's client request headers.
1. Migrate NGINX from its original node group to a new, updated node
group with zero downtime to it or the echoserver during load testing.
1. And lastly, once migration has completed, we'll drain, delete, and remove the
original node group from Kubernetes and AWS.

The code for this tutorial is available on [GitHub][example-gh].

<p align="center"><img src="/images/docs/reference/kubernetes/eks-update-nodegroups.svg" width="650"></p>

{{< aws-eks-prereqs >}}

## Table of Contents

- [Initialize the Pulumi Project](#initialize-the-pulumi-project)
- [Create an EKS Cluster & Deploy Workloads](#create-an-eks-cluster-deploy-workloads)
- [Access the Workloads](#access-the-workloads)
- [...But First, Let's Talk About Resource Updates](#but-first-let-s-talk-about-resource-updates)
  * [Pulumi's Approach: `create-before-delete`](#pulumi-s-approach-create-before-delete)
  * [Kubernetes Workloads: High-Availability (HA) & Rolling Updates](#kubernetes-workloads-high-availability-ha-rolling-updates)
- [The Great Migration](#the-great-migration)
  * [Step 0: Launch Load Tests](#step-0-launch-load-tests)
  * [Step 1: Creating the new `4xlarge` Node Group](#step-1-creating-the-new-4xlarge-node-group)
  * [Step 2: Retargeting NGINX at the `4xlarge` Node Group](#step-2-retargeting-nginx-at-the-4xlarge-node-group)
  * [Step 3: Decomissioning the `2xlarge` Node Group](#step-3-decomissioning-the-2xlarge-node-group)
- [Clean Up](#clean-up)
- [Summary](#summary)

## Initialize the Pulumi Project

1.  Start by cloning the [example][example-gh] to your local machine:

    ```bash
    git clone https://github.com/pulumi/pulumi-eks
    cd nodejs/eks/examples/migrate-nodegroups
    ```

1.  Install the dependencies:

    ```
    npm install
    ```

1.  Create a new Pulumi [stack][stack]:

    ```
    pulumi stack init update-nodegroups-dev
    ```

1. Set the Pulumi [configuration][pulumi-config] variables for the project:

    ```bash
    // Any valid EKS region. AMIs used must match this region.

    pulumi config set aws:region us-west-2
    ```

## Create an EKS Cluster & Deploy Workloads

Perform the cluster and workload deployment:

```bash
pulumi up
```

The update will create the following resources in AWS:

* A VPC in our region, with public & private subnets across all of the region's availability zones.
* The IAM Roles & Instance Profiles for each node group.
* An EKS cluster with `v1.13` of Kubernetes, with settings that include:
    * Private worker nodes
    * Resource tagging, and
    * Omission of the default node group in favor of our own managed worker
      node groups.
* A standard `t2.medium` worker node group using the recent `v1.13.7` worker [AMI][eks-amis].
	* For general purpose workloads, such as the `echoserver`.
* A 2xlarge `t3.2xlarge` worker node group using the previous `v1.12.7` worker [AMI][eks-amis].
	* For larger, intensive workloads such as the [NGINX Ingress Controller][ingress-nginx].

Once the update is complete, verify the cluster, node groups, and Pods are up
and running:

```bash
pulumi stack output kubeconfig > kubeconfig.json && export KUBECONFIG=$PWD/kubeconfig.json
kubectl get nodes -o wide --show-labels -l beta.kubernetes.io/instance-type=t2.medium
kubectl get nodes -o wide --show-labels -l beta.kubernetes.io/instance-type=t3.2xlarge
kubectl get pods --all-namespaces -o wide --show-labels
```

## Access the Workloads

With the deployment from the previous step completed, we can leverage Pulumi
[stack outputs][stack-outputs] to retrieve the public endpoint of the
AWS Classic Load Balancer exported by our Pulumi program.

This load balancer fronts the NGINX Ingress Controller, which in turn
manages the ingress for the `echoserver`:

```bash
$ pulumi stack output nginxServiceUrl
a3a6ae14f9e6d11e99ea4023e81f316e-1155138699.us-east-2.elb.amazonaws.com
```

After the load balancer is provisioned and resolvable on AWS
(takes ~2-3 minutes post-deployment), let's access the `echoserver` behind
NGINX:

```bash
export LB=`pulumi stack output nginxServiceUrl`
curl -Lv -H "Host: apps.example.com" $LB/echoserver
```

> *Note*: If we used a Kubernetes DNS manager such as
[external-dns][external-dns], we could simply use the FQDN instead of the load
balancer's endpoint with custom host headers.

On success, we'll see output similar to the following:

```bash
...
Hostname: echoserver-xlvnkcng-74648fd4dd-8jck4

Pod Information:
	-no pod information available-

Server values:
	server_version=nginx: 1.13.0 - lua: 10008

Request Information:
	client_address=172.16.247.199
	method=GET
	real path=/echoserver
	query=
	request_version=1.1
	request_uri=http://apps.example.com:8080/echoserver

Request Headers:
	accept=*/*
	host=apps.example.com
	user-agent=curl/7.58.0
	x-forwarded-for=172.16.200.230
	x-forwarded-host=apps.example.com
	x-forwarded-port=80
	x-forwarded-proto=http
	x-original-uri=/echoserver
	x-real-ip=172.16.200.230
	x-request-id=b874c43cc675d28c68a0c39069799b10
	x-scheme=http

Request Body:
	-no body in request-
```

With our apps now verified as working and accessible, we're ready to see how we
can leverage Kubernetes and Pulumi to update our worker node groups, and 
migrate our workloads.

## ...But First, Let's Talk About Resource Updates

### Pulumi's Approach: `create-before-delete`

Resources under management in Pulumi are [uniquely named][pulumi-urn] using a
Universal Resource Name (URN) to identify and track resources across program
deployments and state changes. Additionally, a [random postfix][pulumi-auto-naming] is added to
resource instantiations to avoid collisions, and to assist in
zero-downtime replacements. This is achieved by using a **create-before-delete**
approach in the Pulumi programming model where:

* First, the new version of the resource is created,
* Then, any references to the original resource are updated to point
to the new resource,
* And lastly, on success of the updates, the original resource is deleted.

These semantics allow Pulumi to replace resources using a blue/green deployment
strategy by default, and this works for most scenarios.

There are situations, as with many Kubernetes workloads, where the intrinsic
details of Pod updates and replacements are encoded in Kubernetes API resources
and scheduling predicates that are managed by the control plane for high-availability.
These management constructs are exempt from the Pulumi engine and require granular handling of
the blue/green deployment of the Pods, and the underlying cloud provider 
resources that Kubernetes depends on.

In these specific cases, we can stand up overlapping copies of the
infrastruture stack in the Kubernetes layers and in the cloud provider for
as long as we need to guarantee a smooth transition. This affords us the 
ability to administer proper migrations in a blue/green strategy catered to
our use-case, and scenarios like this are completely possible to achieve in Pulumi.

### Kubernetes Workloads: High-Availability (HA) & Rolling Updates

Before we perform an update to our node groups and migrate our workloads, it's
important to consider how equipped our workloads are for rolling updates, if they
employ high availability, and if they can gracefully terminate within the Kubernetes [Pod
lifecycle][pod-lifecycle].

Kubernetes has many knobs and levers through API resources to describe your 
app's requirements, runtime settings, and update strategy. But it requires
you take the necessary steps to refactor your apps to execute, and survive
in a Kubernetes cluster.

In our example, both NGINX and the `echoserver` are
[stateless][k8s-run-stateless] apps, which means that there is no active state
to manage nor persist across running sessions. Both apps are also equipped to
gracefully terminate, and process [SIGTERM][sigterm] where necessary.
Kubernetes is great at managing stateless apps because it's straight-forward to
run, delete, and most importantly, cleanly recovers in the case of failure
or lack of quorum, as availability is as simple as spinning up another replica.

If you're managing stateful apps, you should consider leveraging
[StatefulSets][statefulsets], as they provide each Pod in the set with a unique
identity and index to employ a systematic approach and ordering to the app's
lifecycle. With that said, Kuberenetes has no insight into the details of your
workload, nor how it operates - this is purposefully done by design.
The work to fully run and operate stateful apps in Kubernetes ultimately lies
with the app developers, operators, and administrators to bake-in and
fine-tune the necessary resilience into the workload and its dependencies
to safely apply updates, and terminate as expected.

As you'll see shortly, we've taken the liberty of using the various
capabilities that Kubernetes offers for HA and rolling updates to support
migration of the NGINX Ingress Controller example.

We highly encourage you to check out the references below to help your
apps achieve proper Kubernetes and production readiness. 

#### References

- [Deployments][k8s-deployments]
- [PodDisruptionBudget][k8s-pdb]
- [Node Selectors & Affinity][k8s-node-affinity]
- [Pod Affinity & Anti-Affinity][k8s-pod-affinity]
- [Service Selectors][k8s-service]
- [Readiness & Liveness Probes][k8s-probes]
- [Lifecycle Hooks][k8s-hooks]
- [Termation Grace Period & Graceful Shutdown][k8s-graceful-shutdown]
- [Pod Termination Lifecycle][pod-lifecycle]
- [Zero-downtime Deployment in Kubernetes with Jenkins][k8s-jenkins]

## The Great Migration

Upon initial deployment of our workload stack, the `echoserver` will ultimately land
on the Standard node group we've created, given our lack of specificity
for where the Kubernetes scheduler should place it. The NGINX Ingress Controller
instead, specifically targets the `2xlarge` node group given its heftier specs.
Both scheduling choices were made with the intent of segmenting our workloads
by use-case, and performance requirements we've identified.

In our [inital update][initial-update] we selected that our EKS 
control plane run on `v1.13.7` of Kubernetes, that the Standard node group
use `v1.13.7` of the [EKS-optimized][eks-amis] worker AMIs, and that the
`2xlarge` node group use `v1.12.7` workers.

Since we do not want our `2xlarge` workers to drift too far apart from the control
plane's version to cause [unsupported skew][version-skew], and we
ultimately realize that the `2xlarge` node group may not suffice for a
large, anticipated request load, we've decided to update various settings of 
the node group for NGINX.

The node group that NGINX will select and target will go from:

  * Using `v1.12.7`of Kubernetes, in a pool of (3) `t3.2xlarge` worker instances ->
  * Using `v1.13.7`of Kubernetes, in a pool of (5) `c5.4xlarge` worker instances.

We will also be using the [`hey`][hey-load-testing] load testing tool through
out our migration to consistently hit the endpoint and path for the
`echoserver` behind NGINX to ensure we acheive zero-downtime.

### Step 0: Launch Load Tests

As we migrate NGINX from the `2xlarge` -> `4xlarge` node group, we'll kick off
a load testing script against the endpoint and path of the `echoserver` on our
cluster.

You can install the [`hey`][hey-load-testing] load testing tool locally to your
machine by doing the following:

```bash
go get -u github.com/rakyll/hey
```

Using the `LB` environment variable previously defined in the 
`pulumi stack output` for NGINX's AWS load balancer, and a helper script in
`./scripts`, we'll load test for many iterations of 50,000 requests, with 100
concurrent requests at a time, e.g. run testing across 75 iterations:

```bash
./scripts/load-testing.sh $LB 75
```

> **Note**: Given the large number of requests being generated during load
> testing (millions), a seperate machine for testing would be best suited
for overall network performance and throughput on your client.

### Step 1: Creating the new `4xlarge` Node Group

Next, we'll create a new node group in AWS using Pulumi for the `4xlarge` node
group. This is as simple as defining a new node group in our program:

<p align="center"><img src="/images/docs/reference/kubernetes/ng-4xlarge.svg" width="650"/></p>

You can make this change yourself in your `index.ts` by running the following: 

```bash
cp steps/step1/index.ts index.ts
pulumi up
```

Once the update is complete, verify the new `c5.4xlarge` node group is up and running:

```bash
kubectl get nodes -o wide --show-labels -l beta.kubernetes.io/instance-type=c5.4xlarge
```

### Step 2: Retargeting NGINX at the `4xlarge` Node Group

Now, we'll retarget the NGINX Service away from the `2xlarge` node group over to the
`4xlarge` node group, by changing its node selector scheduling terms:

<p align="center"><img src="/images/docs/reference/kubernetes/target-ng-4xlarge.svg" width="650"/></p>

You can make this change yourself in your `index.ts` by running the following: 

```bash
cp steps/step2/index.ts index.ts
pulumi up
```

> **Note:** Given that the termination process must gracefully shutdown, and process
all in-flight requests, this may take a few minutes to complete.

Once the update is complete, verify the NGINX pods are now running in the new
`4xlarge` node group by retrieving its Pods, the `c5.4xlarge` nodes, and
noticing that the Pods are in fact running on updated nodes.

```bash
kubectl get pods --all-namespaces -o wide --show-labels -l app=nginx-ing-cntlr
kubectl get nodes -o wide --show-labels -l beta.kubernetes.io/instance-type=c5.4xlarge
```

You should also notice a linear up-tick in **requests per second** in the
load testing results, due to the more capable `c5.4xlarge` worker instances
being used.

### Step 3: Decomissioning the `2xlarge` Node Group

With NGINX validated to be up and running on the `4xlarge` node group, we can
now commence the decomissioning of the original `2xlarge` node group no
longer in use.

First, we'll instruct the Kubernetes APIServer to drain the `2xlarge` node group using
`kubectl drain`. We've added a helper script to facilitate this process:

```bash
./scripts/drain-t3.2xlarge-nodes.sh
```

Once draining is complete, let's instruct the Kubernetes APIServer to delete
and remove the nodes from the cluster using `kubectl delete node`:

```bash
./scripts/delete-t3.2xlarge-nodes.sh
```

After node deletion from the cluster has completed, and we've verified that the
instances backing our load balancer are up-to-date and no longer include the
`2xlarge` node group, we can remove its definition from our Pulumi program:

<p align="center"><img src="/images/docs/reference/kubernetes/remove-ng-2xlarge.svg" width="650"/></p>

You can make this change yourself in your `index.ts` by running the following: 

```bash
cp steps/step3/index.ts index.ts
pulumi up
```

If we've executed all of the steps as follows, then we'll have successfully migrated
NGINX from the `2xlarge` node group to the `4xlarge` group with
zero downtime to it or the `echoserver`, and removed the
`2xlarge` node group completely from Kubernetes and AWS.

We can also verify the load testing results to validate that our requests have all 
returned with `HTTP 200` status codes through out the entire migration
process.

## Clean Up

Run the following command to tear down the resources that are part of our stack.

1.  Run `pulumi destroy` to tear down all resources.  You'll be prompted to make sure you really want to delete these resources.

1.  To delete the stack itself, run `pulumi stack rm`. Note that this command deletes all deployment history from the Pulumi Console and cannot be undone.

## Summary

In this tutorial, we saw how to use Pulumi to launch a managed Kubernetes 
cluster on AWS EKS with active workloads. Then, we performed a migration
of the apps and underlying cloud provider resources over to new, updated
resources with zero downtime.

Leveraging Kubernetes control plane with Pulumi's Infrastrcture-as-Code (IaC)
semantics allows us to administer over many kinds of update scenarios with
ease. This provides the ability to manage clusters with real code that can
be tracked in version control systems, and deployed as a series of
Pulumi updates through CI/CD.

For a follow-up example on how to further use Pulumi to create Kubernetes
clusters, or deploy workloads to a cluster, check out the rest of the
[Kubernetes tutorials]({{< relref "/docs/reference/tutorials/kubernetes" >}}).

[example-gh]: https://github.com/pulumi/pulumi-eks/tree/master/nodejs/eks/examples/migrate-nodegroups
[stack]: https://pulumi.io/reference/stack
[eks-amis]: https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html
[ingress-nginx]: https://github.com/kubernetes/ingress-nginx
[echoserver]: https://github.com/kubernetes-retired/contrib/blob/master/ingress/echoheaders/echo-app.yaml
[stack-outputs]: https://www.pulumi.com/docs/reference/programming-model#stack-outputs
[pulumi-config]: https://pulumi.io/reference/config
[external-dns]: https://github.com/kubernetes-incubator/external-dns
[pod-lifecycle]: https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods
[k8s-run-stateless]: https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment
[statefulsets]: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset
[k8s-deployments]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment
[k8s-pdb]: https://kubernetes.io/docs/tasks/run-application/configure-pdb
[k8s-node-affinity]: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
[k8s-pod-affinity]: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity-beta-feature
[k8s-service]: https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service
[k8s-probes]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes
[k8s-hooks]: https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks
[k8s-graceful-shutdown]: https://pracucci.com/graceful-shutdown-of-kubernetes-pods.html
[k8s-jenkins]: https://kubernetes.io/blog/2018/04/30/zero-downtime-deployment-kubernetes-jenkins
[eks-amis]: https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html
[version-skew]: https://kubernetes.io/docs/setup/release/version-skew-policy/#supported-version-skew
[pulumi-urn]: https://www.pulumi.com/docs/reference/programming-model/#urns
[pulumi-auto-naming]: https://www.pulumi.com/docs/reference/programming-model/#autonaming
[sigterm]: https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html
[initial-update]: #create-an-eks-cluster-deploy-workloads
[hey-load-testing]: https://github.com/rakyll/hey
