---
layout: post
title:  "Reduce Kubernetes Cluster Costs on AWS"
date:   2020-06-01 18:31:00 +0800
categories: kubernetes
---

When you spun up a Kubernetes cluster for your own personal blog, you are on a
tight budget. Kubernetes cluster can blow a hole in your pocket if you don't
pay attention to the AWS services it uses under the hood.

For this blog, the Kubernetes cluster was provisioned using [kops][kops-repo]. There
are settings you could change only during creation and there are also things
you could change post creation. For the former point, you better plan it early
before you create the cluster.

Mainly there are 2 things that you need to pay attention for costs reduction.
1. EC2 instances.
2. EBS volumes.

## EC2 Instances

The minimum number of instances you need to run a proper cluster is 3 instances,
1 master and 2 nodes.
By default, the EC2 instance will be an on-demand instance so to reduce the cost,
you need to change to AWS Spot instances. For this, you have to explicitly set
the maximum price for your master and nodes instances. This settings can be done
either during creation or post creation.

### New Cluster

If you want to set it during cluster creation, you will need to run few commands.

First you will need to generate the cluster yaml file.

```
kops create cluster \
  --name=k8s.blog.taufek.dev \
  --state=s3://k8s.blog.taufek.dev.state \
  --zones=ap-southeast-1a,ap-southeast-1b,ap-southeast-1c \
  --dns-zone=k8s.blog.taufek.dev \
  --dry-run \
  --output yaml | tee k8s.blog.taufek.dev.yml
```

Then edit the `k8s.blog.taufek.dev.yml` file by adding `maxPrice: "#.##"` for
both master and nodes settings.

{% highlight yaml %}
{% raw %}
apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  ...
  name: master-ap-southeast-1a
spec:
  ...
  maxPrice: "0.10" # Add this line
  maxSize: 1
  ...
---
apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  ...
  name: nodes
spec:
  ...
  maxPrice: "0.10" # Add this line
  maxSize: 2
  ...
{% endraw %}
{% endhighlight %}

Then run below command to create the cluster

```
kops create -f k8s.blog.taufek.dev.yml --state=s3://k8s.blog.taufek.dev.state
kops create secret --name k8s.blog.taufek.dev sshpublickey admin -i ~/.ssh/id_rsa.pub
kops update cluster k8s.taufek.dev --yes --state=s3://k8s.blog.taufek.dev.state
```

Note: Before running above to create the cluster, you might want to read
`EBS Volumes` section below because part of the setting is not easily change
after the cluster is created.

### Existing Cluster

If you already have a running cluster, you can edit the cluster settings. First,
run below command to open up the Cluster object.

```
kops edit cluster --state=s3://k8s.blog.taufek.dev.state
```

Then it will open up similar cluster yaml file in creation step above. Add the
`maxPrice` fields and save the file.

Then run below to push the change to your cluster.

```
kops update cluster --yes --state=s3://k8s.blog.taufek.dev.state
kops rolling-update cluster --yes --state=s3://k8s.blog.taufek.dev.state
```

Run below to know when your cluster is ready

```
kops validate cluster --state=s3://k8s.blog.taufek.dev.state
```

## EBS Volumes

`kops` provisions 1 volume for each instances in your cluster and 2 volumes
for etcd cluster. In my 1 master and 2 nodes cluster, I will be given 5 volumes.
By default, the volumes size are crazily big, at least too big for running
my blog site. By default, master volume size is 64G, nodes is 128G and etcd is 2x20GB
volumes.

For master and nodes volume size, they can be change during creation and post creation.
Unfortunately, `kops`, does not allow to change etcd volume size during creation.

### New Cluster

First you will need to generate the cluster yaml file.

```
kops create cluster \
  --name=k8s.blog.taufek.dev \
  --state=s3://k8s.blog.taufek.dev.state \
  --zones=ap-southeast-1a,ap-southeast-1b,ap-southeast-1c \
  --master-volume-size=8 \
  --node-volume-size=8 \
  --dns-zone=k8s.blog.taufek.dev \
  --dry-run \
  --output yaml | tee k8s.blog.taufek.dev.yml
```

For master and nodes volume, you could specify via the command line above but
for etcd cluster, you will need to modify the cluster yaml file manually.

Then edit the `k8s.blog.taufek.dev.yml` file.

{% highlight yaml %}
{% raw %}
apiVersion: kops.k8s.io/v1alpha2
kind: Cluster
...
spec:
  ...
  etcdClusters:
  - cpuRequest: 200m
    etcdMembers:
    - instanceGroup: master-ap-southeast-1a
      name: a
      volumeSize: 8 # Add this volume in GB
    ...
  - cpuRequest: 100m
    etcdMembers:
    - instanceGroup: master-ap-southeast-1a
      name: a
      volumeSize: 8 # Add this volume in GB
    ...
{% endraw %}
{% endhighlight %}

Then run below command to create the cluster

```
kops create -f k8s.blog.taufek.dev.yml --state=s3://k8s.blog.taufek.dev.state
kops create secret --name k8s.blog.taufek.dev sshpublickey admin -i ~/.ssh/id_rsa.pub
kops update cluster k8s.taufek.dev --yes --state=s3://k8s.blog.taufek.dev.state
```

### Existing Cluster

Run below command to open up the Cluster object.

```
kops edit cluster --state=s3://k8s.blog.taufek.dev.state
```

Within the Cluster object, set below to configure master and nodes volumes size:

{% highlight yaml %}
{% raw %}
apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  ...
  name: master-ap-southeast-1a
spec:
  ...
  role: Master
  rootVolumeSize: 8 # Add this value in GB

---

apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  ...
  name: nodes
spec:
  ...
  role: Node
  rootVolumeSize: 8 # Add this value in GB
{% endraw %}
{% endhighlight %}

Then run below to push the change to your cluster.

```
kops update cluster --yes --state=s3://k8s.blog.taufek.dev.state
kops rolling-update cluster --yes --state=s3://k8s.blog.taufek.dev.state
```

For etcd volumes, you can't use `kops` to modify the volume size. But there are workaround
mentioned in this [article][reduce-existing-etcd-volumes]. I personally used this
workaround to reduce my etcd cluster volumes.

## Conclusions

Before I carried out above cost optimization steps, below were the Cluster setup:
1. 1 master instance with on-demand instance
2. 2 nodes instances with on-demand instance
3. 1 master EBS volume with 64G volume size
4. 2 nodes EBS volume with 128G volume size each
5. 2 etcd EBS volume with 20G volume size each

Now the Cluster setup looks like below:
1. 1 master instance with spot instance (up to 90% price reduction)
2. 2 nodes instances with spot instance (up to 90% price reduction)
3. 1 master EBS volume with 8G volume size
4. 2 nodes EBS volume with 8G volume size each
5. 2 etcd EBS volume with 8G volume size each

At the point of writing this post, my Cluster is just 2 days old. I will update 
this space about the monthly costs once I have to pay the AWS bill at the end of this month.


[kops-repo]: https://github.com/kubernetes/kops
[reduce-existing-etcd-volumes]: https://medium.com/@int128/resize-etcd-volumes-on-kops-e499b3936383
