# How to Run a GKE Cluster on the Cheap (As Low as $24 Per Month)

What if I told you that you could run a 3 node, 6 core, GKE cluster at over a 90% discount, or only about $24 per month? In this Git repository, you can use Terraform to deploy a GKE cluster with all of the cost savings maneuvers in place to get you up and running with GKE on the cheap.

**Warning: Google Cloud only gives you 1 free GKE control plane. If you run more than 1 GKE cluster, you will incur $74.40 per month for each control plane no matter what!**

### [See this blog post for a detailed explanation: TODO](http://)

This solution is primarily for people learning Kubernetes and GKE, or just people tinkering with GKE that don't want to pay full price. GKE is expensive to the individual and this solution makes it even cheaper than low-cost Kubernetes services like Civo (provided that you only run 1 cluster).

The solution, and its cost savings, is scalable to a larger cluster. It is conceivable that cost-conscious organizations may be interested in the techniques used in this solution as well. Just be aware of the drawbacks! See the blog post for more information.

## Prepare to Deploy the GKE Cluster

It is assumed that you already have `gcloud` installed and initialized for use with your project. If not, follow the instructions on this [page](https://cloud.google.com/sdk/docs/install).

### Install Terraform

Terraform is required to run the deployment on your local machine. On a Mac you can use Homebrew by running these commands:

```
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

### Configure Terraform

Change `terraform.tfvars` to use your Google Cloud `PROJECT_ID`. You can find your `PROJECT_ID` by running this command:

```
gcloud projects list
```

You should also update your current project for `gcloud` if it's not set to the one that you intend to deploy the cluster to:

```
gcloud config set project REPLACE_WITH_YOUR_PROJECT_ID
```
You may also choose to change the region you choose to deploy. Each GCP region has different pricing for VM Spot instances. See this [page](https://cloud.google.com/compute/vm-instance-pricin) for pricing details.

## Deploy the GKE Cluster

Now you can run Terraform to deploy GKE and all of the supporting components needed for this solution to work.

```
cd terraform
terraform init
terraform apply --auto-approve
```

Terraform will take several minutes to run. Please be patient! A lot of things are being provisioned and installed beyond just GKE.

Next, you must deploy the Petstore sample application and a `VirtualService` to route traffic to that application:

```
kubectl apply -f ../petstore.yaml
kubectl apply -f ../virtualservice.yaml
```

Get the IP Address of the load balancer for running the `curl` command to verify deployment. Change the `my-static-ip` if it was changed in the `terraform.tfvars`
```
ipaddress=$(gcloud compute addresses describe my-static-ip --format="value(address)")
```

Run the curl command. You should see JSON data from the Petstore application.
```
curl http://$ipaddress

[{"id":1,"name":"Dog","status":"available"},{"id":2,"name":"Cat","status":"pending"}]
```

## Tear Down the GKE Cluster

When you are done, you can remove everything from Google Cloud with this command:

```
terraform destroy
```

## How Does It Work?

The overall solution is a bit complex and does use some Beta features of Google Cloud. However, the solution has been implemented in Terraform so it's easy to deploy, even with the many components and configurations required.

These are the main parts of the solution to achieve a high level of cost savings:

1. Use a private, [VPC-native](https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg#create_a-native_cluster), [GKE cluster using only Spot VM instances](https://cloud.google.com/kubernetes-engine/docs/concepts/spot-vms) as the cluster nodes. This will save you up to 90% on the cost of VMs, depending on the GCP region. This could save you up to $150 per month for GKE node VMs for a 3 node, 6 core cluster.

1. Use a [Regional (rather than Global) HTTP Load Balancer](https://cloud.google.com/load-balancing/docs/https) which is currently free as a Beta preview. Additional costs may be incurred in the future. This currently saves you $18.26 per month.

    a. Attach the [Regional HTTP Load Balancer to a standalone NEG](https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg#how_to) to route traffic directly from the Load Balancer to the GKE ingress gateway pods to achieve container-native load balancing.

1. Use [Gloo Edge](https://github.com/solo-io/gloo) for an Envoy-based GKE ingress gateway that provides advanced routing capabilities to services running in the cluster. Gloo Edge also provides reliable traffic routing to mitigate potential errors resulting from Spot VM node shutdowns. Gloo Edge is open source and free to use.

1. The [first GKE control plane is free](https://cloud.google.com/kubernetes-engine/pricing#cluster_management_fee_and_free_tier). This currently saves $74.40 per month.

1. Deploy a [Cloud NAT](https://cloud.google.com/nat/docs/overview) gateway to enable  egress from the private GKE cluster. Additional cost is about [$1 per VM per month](https://cloud.google.com/nat/pricing) plus $0.045 per GB for data processing on egress traffic (which is primarily for pulling external Docker images). If you pull a lot of your own images, using [Artifact Registry](https://cloud.google.com/artifact-registry) would be advisable.

Terraform configs (`.tf`) are commented with specific details and references to explain how the deployment works and why. Also see the blog post for specifics.

## Next Steps For Using Your Cheap GKE Cluster

### Understanding Gloo Edge

[Gloo Edge](https://docs.solo.io/gloo-edge/master/) provides powerful traffic routing capabilities that go far beyond the standard [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/). As Gloo Edge uses Envoy, capabilities such as retries help improve the resiliency of routing to applications in your cluster that is using Spot VM node instances.

It's beneficial, but not required, to [install `glooctl`](https://docs.solo.io/gloo-edge/master/installation/glooctl_setup/) to work with Gloo Edge.

### Update Config and Deploy Another Application

Next you should proceed with:

1. Examining `virtualservice.yaml` and [understand how it works](https://docs.solo.io/gloo-edge/master/introduction/traffic_management/).
1. Deploying your own application onto your new Kubernetes cluster.
1. Modifying `virtualservice.yaml` to use your application's upstream. You can view upstreams with `glooctl get upstream`. Make sure the application is available at `/` or the Load Balancer health checks will fail. You may choose to rewrite the path as is done in `virtualservice.yaml` or change the `regional-l7-xlb-map-http` and `l7-xlb-basic-check-http` in `load-balancer.tf` to use a path other than `"/"`.

### Use Anti-Affinity Rules for Application Resilency

Spot VM nodes can be shut down at any time. You should strive to run 2 replicas of your pods with `podAntiAffinity` set. See `petstore.yaml` for an example of this.

### Use Retries for Application Resilency

When a Spot VM node goes down, there may be traffic in route to the pods on that node. This may result in HTTP errors. As such, it's best to implement a retry mechanism that  allows HTTP requests to be resent to the 2nd instance of your application. See `virtualservice.yaml` for an example of implementing retries with Gloo Edge.

If you have a microservice architecture with microservices deployed across nodes, Istio service mesh (not provided in this solution) would allow you to implement retries between services. This topic might be covered in a future blog post in the context of this solution.

## Frequently Asked Questions

1. *How long do the Spot VM nodes run?*

    The longest period of time that I have seen is 23 days in `us-west4`. Most typically will run for several days.

2. *Will only one Spot VM node be replaced at a time?*

    There is no guaranteee when nodes or how many of the nodes will be replaced. Here is an example where 2 of the 3 nodes were replaced at the same time, as they both have the same age. Unfortunately, if your application workload only resided on those 2 nodes, it would have been temporarily offline until new pods spun up to replace them.

```
NAME                                        STATUS   ROLES    AGE     VERSION
gke-my-cluster-default-pool-8dc7ce28-dl7h   Ready    <none>   4h14m   v1.21.5-gke.1302
gke-my-cluster-default-pool-8dc7ce28-k3d3   Ready    <none>   4h14m   v1.21.5-gke.1302
gke-my-cluster-default-pool-8dc7ce28-cpkd   Ready    <none>   23d     v1.21.5-gke.1302
```

3. *How can I guarantee that my applications will stay online if multiple Spot VM nodes are replaced at the same time?*

    Having a 100% Spot VM node cluster is not recommended for critical production workloads. You might consider having 2 node pools and spread the workloads across both non-spot and spot nodes. You may consider using [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/) where the `topologyKey` could be `cloud.google.com/gke-spot`. Of course, using non-spot nodes will result in additional cost, but having a portion of your cluster be spot nodes will save significant amounts of money compared to an entirely non-spot cluster.

    Generally speaking, the solution presented in this GitHub repo is for non-production purposes only.

4. *Why do I have `Terminated` pods and how can I get rid of them?*

    When a Spot VM node shuts down, pods residing on the node will move to a `Terminated` state. Kubernetes [doesn't currently garbage collect](https://github.com/kubernetes/kubernetes/issues/99986) these `Terminated`/`Failed` pods. You may periodically delete these pods with the following command:

```
kubectl delete pods --field-selector status.phase=Failed --all-namespaces
```

5. *What GCP regions are compatible with this solution?*

    This solution has been tried successfully with `us-west4` and `asia-east2`, both of which have very low Spot VM instance prices. It was not compatible with `europe-central2` due to lack of the Standard networking tier. Otherwise, the best way to know is to give it a try and see if the deployment is successful.

6. *What about HTTPS/TLS termination for the Regional HTTP Load Balancer?*

    Implementing HTTPS (including a redirect) is possible, but not currently incorporated into this solution. There is a GitHub issue open [here](https://github.com/murphye/cheap-gke-cluster/issues/1) with details. 

7. *Can I do TCP passthrough rather than HTTP for the Regional Load Balancer?*

    This is not currently possible with Google's managed proxy service, but is possible with Envoy, should Google decide to support it in the future.

8. *Can I use an Ingress Controller implementation such as NGINX instead of Gloo Edge?*

    While not provided in this solution, yes you can use an alternative, but Gloo Edge/Envoy is ideally suited for this use case where a high level of resiliency is needed due to the use of Spot VM cluster nodes.

9. *What if I have stateful workloads such as a PostgreSQL database?*

    Generally speaking, you should consider having a 2nd cluster node pool dedicated to stateful workloads. However, if you're dead set on using Spot VM nodes, you might consider adding resilence to your database deployment with replicated container storage such as Portworx to [implement a form of HA with PostgreSQL](https://portworx.com/blog/ha-postgresql-kubernetes/). You might also consider database alternatives such as [CockroachDB on Kubernetes](https://www.cockroachlabs.com/docs/v21.2/deploy-cockroachdb-with-kubernetes-insecure). Or, **preferably**, [create a regional persistent disk for PostgreSQL](https://cloud.google.com/architecture/deploying-highly-available-postgresql-with-gke) for deploying highly available PostgreSQL with GKE. This will not bound any of your data to the cluster nodes as the `PersistentVolumeClaim` will be external to the cluster.
    
   Any applications that you have connecting to the database must be able to automatically reconnect should the connection be lost temporarily. If this isn't possible, you should implement a `livenessProbe` for the application based on database connectivity so the application can be redeployed when the `livenessProbe` fails. 
   
   As an example, this [article](https://medium.com/@ievgen.degtiarenko/liveness-and-readiness-probes-for-spring-boot-apps-running-in-k8s-fd43a9a9dca8) explains how to do this with Spring Boot Actuator and PostgreSQL. You may also combine this with [HikariCP](https://github.com/brettwooldridge/HikariCP#essentials) for database connection pooling. Should HikariCP not be able to reconnect, eventually the `livenessProbe` will fail as well causing the Spring Boot application to be redeployed which may be desirable in an unrecoverable database connection scenario. As a best practice, a TCP socket timeout should be set to a minimum of 30 seconds. More information is available [here](https://github.com/brettwooldridge/HikariCP/issues/1056) and [here](https://www.theguardian.com/info/2019/dec/02/faster-postgresql-connection-recovery).