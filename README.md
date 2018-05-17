# Production Grade Kubernetes Clusters on AWS with Kops
In this talk, we'll live-demo bringing up a new production grade Kubernetes cluster in AWS. To do this, we'll use a tool called kops. We'll cover how kops works and why I recommend using it (at least until EKS is generally available). We'll also talk about what makes a cluster production grade and how to integrate a new Kubernetes cluster with your existing infrastructure. This will be a DevOps focused talk and we'll dig into AWS networking, Terraform, and how to manage your core Kubernetes manifests.

This talk was given at the [Barcelona Kubernetes Meetup](https://www.meetup.com/Kubernetes-Barcelona/events/250572122/)
on May 16th, 2018.

Slides: https://speakerdeck.com/lkysow/production-grade-kubernetes-clusters-on-aws-with-kops

# Basic Kops Demo

## Prerequisites
* Follow https://github.com/kubernetes/kops/blob/master/docs/aws.md to set up an IAM
user specific to kops or use one you already have set up if it's got the right permissions.
* We're going to use kops "gossip DNS" configuration to start so don't need to create any zones.
In a production-ready set up I'd recommend using an actual DNS zone so you get proper DNS records.
* Create an S3 bucket for kops to store its state.

## Create Cluster
```bash
export CLUSTER_NAME=kube.lkysow.k8s.local
export KOPS_STATE_STORE=s3://lkysow-kops
# Push the desired state to S3.
kops create cluster \
    --zones us-east-1e \
    "$CLUSTER_NAME"
# Tell kops to actually create the cluster.
kops update cluster kube.lkysow.k8s.local --state=$KOPS_STATE_STORE --yes
```

Kops will start spinning up resources immediately.

Wait until the cluster is ready using:
```
watch kubectl get componentstatus
# When ready will return:
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```

Validate that all nodes have joined the cluster with:
```
kops validate cluster --state $KOPS_STATE_STORE
```

You now have a Kubernetes cluster! However, it's not production ready.

# Make it production ready.
## Multi-master, multi-az and private subnets
The basic demo above spins up the cluster in a single availability zone, with a
single master, and with the instances in public subnets. The API ELB is also
exposed publicly.

This command will create a cluster in three availability zones with three master,
in private subnets and with the API ELB only whitelisted to your current IP.
You can also set the ELB to be internal if you have a VPN into your VPC.
```bash
EXTERNAL_IP=$(dig +short myip.opendns.com @resolver1.opendns.com)
ZONES="us-east-1c,us-east-1d,us-east-1e"
kops create cluster \
    --node-count 1 \
    --cloud aws \
    --zones $ZONES \
    --master-zones $ZONES \
    --node-size t2.medium \
    --master-size t2.medium \
    --networking flannel-vxlan \
    --admin-access "$EXTERNAL_IP/32" \
    --ssh-access "$EXTERNAL_IP/32" \
    --state s3://lkysow-kops \
    --topology private \
    kube-ha.lkysow.k8s.local
kops update cluster kube-ha.lkysow.k8s.local --yes --state s3://lkysow-kops
```

## Kubernetes Version
Unfortunately, there's still some changes to make. The default cluster is
on an old Kubernetes version:
```
kubectl version --short
Client Version: v1.10.0
Server Version: v1.9.3
```
As of this writing the latest 1.9.* version is 1.9.7 and the latest major version
is 1.10.2.

Arguably worse, you're on an old etcd version:
```
kubectl get pod -n kube-system | grep etcd
kubectl port-forward <pod id> 4001
curl -k https://localhost:4001/version
{"etcdserver":"2.something","etcdcluster":"2.something"}%
```
Etcd v3 is the latest version and I imagine v2 will soon be deprecated. Starting
your cluster with version 3 will save you a lot of pain!

There is a `kubernetes-version` flag, but no `etcd-version` flag so you need
to edit the cluster spec as YAML and change the `version`. See spec: https://godoc.org/k8s.io/kops/pkg/apis/kops#EtcdClusterSpec
```
kops edit cluster kube-ha.lkysow.k8s.local --state s3://lkysow-kops
```

## Version Control
We've been editing everything through the CLI but there's nothing in version
control and this is hard to automate. Instead, export the cluster as YAML:

```
kops get kube-ha.lkysow.k8s.local --state s3://lkysow-kops -o yaml > cluster.yaml
vim `cluster.yaml`
```

When you're done any changes, use `kops replace` to push your new desired state to S3:
```
kops replace -f cluster.yaml kube-ha.lkysow.k8s.local --state s3://lkysow-kops
```
Apply the changes with:
```
kops update cluster kube-ha.lkysow.k8s.local --state s3://lkysow-kops
```

## Terraform
If you use Terraform (and you should) then you might dislike kops spinning up resources
without corresponding Terraform. Luckily, kops can output Terraform: https://github.com/kubernetes/kops/blob/master/docs/terraform.md

## Create your own VPC
Personally, I like to create the VPC using my own Terraform so I can make changes
more easily, for example adding peering routes, tagging things the way I like, etc.

Luckily, kops supports deploying into an existing VPC: https://github.com/kubernetes/kops/blob/master/docs/run_in_existing_vpc.md

# Conclusion
Kops is a great tool for building Kubernetes clusters however it requires some
important changes from the basic configuration to be production ready.
