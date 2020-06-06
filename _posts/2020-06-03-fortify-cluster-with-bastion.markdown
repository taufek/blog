---
layout: post
title:  "Fortify Kubernetes Cluster with Bastion"
date:   2020-06-06 03:00:00 +0800
categories: kubernetes
hero_src: stormtrooper.jpg
---

In previous [post]({% post_url 2020-06-03-manage-services-with-ingress %}) we looked into Ingress as the front facing entry point for your applications users.
In this post, we will look into Bastion as the backdoor entry point for infrastructure administration users.

Currently, all the EC2 instances in my cluster are on public subnet which all of them has public IP address.
In order to secure the cluster further, we can move our cluster to a private subnet and setup a Bastion instance
as a 'jump' server. Bastion will be the only instance on your public subnet and it has access to the cluster.
We can easily achieve this setup with `kops`.

## Create New Cluster with Bastion

For creating new cluster, you can run below command and your will get a cluster on private network with a Bastion instance ready:

{% highlight bash %}
kops create cluster \
  --name=k8s.blog.taufek.dev \
  --state=s3://k8s.blog.taufek.dev.state \
  --zones=ap-southeast-1a,ap-southeast-1b,ap-southeast-1c \
  --dns-zone=k8s.blog.taufek.dev \
  --topology=private \
  --bastion
  --yes
{% endhighlight %}

Then run below commands:

```
kops update cluster \
  --name kubernetes.taufek.dev \
  --yes \
  --state=s3://k8s.blog.taufek.dev.state
```

In a minute or so you will have your cluster ready. You could run below command to know the status:
```
kops validate cluster --state=s3://k8s.blog.taufek.dev.state
```


## Add Bastion to Existing Cluster

For existing cluster, you will need to to edit your cluster object. Run below command to open up your cluster yaml file

```
kops edit cluster --state=s3://k8s.blog.taufek.dev.state
```

Edit your cluster yml with following settings. You could get the subnets by running `kops` create command with dry-run flag and output to a file.

{% highlight bash %}
  ...
  subnets:
  - cidr: ...
    name: ap-southeast-1a
    type: Private            # change this back Private
    zone: ap-southeast-1a
  - cidr: ...
    name: ap-southeast-1b
    type: Private            # change this back Private
    zone: ap-southeast-1b
  - cidr: ...
    name: ap-southeast-1c
    type: Private            # change this back Private
    zone: ap-southeast-1c
  - cidr: ...                # add these new Utility subnets
    name: utility-ap-southeast-1a
    type: Utility
    zone: ap-southeast-1a
  - cidr: ...
    name: utility-ap-southeast-1b
    type: Utility
    zone: ap-southeast-1b
  - cidr: ...
    name: utility-ap-southeast-1c
    type: Utility
    zone: ap-southeast-1c
  topology:
    bastion:                 # add bastion entry here
      bastionPublicName: bastion.blog.k8s.taufek.dev
    dns:
      type: Public
    masters: private         # change to private
    nodes: private           # change to private
{% endhighlight %}

Then run below to create Bastion instance.

```
kops create instancegroup bastions \
  --role Bastion \
  --subnet ap-southeast-1a,ap-southeast-1b,ap-southeast-1c \
  --state=s3://k8s.blog.taufek.dev.state
```

Run below command to push update to your cluster

```
kops update cluster --yes --state=s3://k8s.blog.taufek.dev.state
kops rolling-update cluster  --state=s3://k8s.blog.taufek.dev.state --yes
```

That's all too it, and just wait for few minute for your cluster to be reconfigured with private subnet.
And you should also be able to ssh into your Bastion instance with following command:

```
ssh -A admin@bastion.blog.taufek.dev
```

Below is how the cluster looks like before without Bastion:

![Cluster without Bastion](/images/cluster_without_bastion.png)

And below is how it looks like now with Bastion:

![Cluster with Bastion](/images/cluster_with_bastion.png)

One thing that made me hesitated to stick with this setup, is the additional ELB it created for Bastion,
which means it will be a significant cost increase in my AWS bill. So I decided to undo this change.

In order to remove Bastion you will need to edit back your cluster yaml file and remove Bastion instance.

Edit your cluster yml with following settings

{% highlight bash %}
  ...
  subnets:
  - cidr: ...
    name: ap-southeast-1a
    type: Public             # change this back to Public
    zone: ap-southeast-1a
  - cidr: ...
    name: ap-southeast-1b
    type: Public             # change this back to Public
    zone: ap-southeast-1b
  - cidr: ...
    name: ap-southeast-1c
    type: Public             # change this back to Public
    zone: ap-southeast-1c
  ...                        # remove the Utility subnets
  topology:
  ...                        # remove bastion
    dns:
      type: Public
    masters: public          # change this back to public
    nodes: public            # change this back to public
{% endhighlight %}

Run below command to push update to your cluster

```
kops update cluster --yes --state=s3://k8s.blog.taufek.dev.state
kops rolling-update cluster  --state=s3://k8s.blog.taufek.dev.state --yes
```

Run below command to remove Bastion instance
```
kops delete instancegroup bastions  --yes --state=s3://k8s.blog.taufek.dev.state
```

## Conclusions

Bastion is not a concept solely for Kubernetes but it's a common practice in fortifying your software infrastructure
even without Kubernetes. I found that by looking at all the AWS services created by `kops`, you could learn
so much about best practices in software infrastructure. I don't have much experience in setting up production
environment but I learn a lot in past couple of weeks by looking at how `kops` wires everything together.

You can't appreciate enough the complexity of services `kops` utility created under the hood with just one command, `kops create ...`.
I can't imagine the hours you have to put in to setup everything manually versus doing it with `kops`.
And it is just not about creating but also managing with this utility. By using this utility not only
you save your time but rest assured that your infrastructure is configured in a correct manner.
