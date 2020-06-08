---
layout: post
title:  "Manage Applications with Ingress"
date:   2020-06-03 23:00:00 +0800
categories: kubernetes
hero_src: road.jpg
---

Ingress is the gate keeper to your cluster. All external
access will go through your Ingress Controller and depending on the rules that you set,
it will be redirected to a particular service.

There are multiple benefits of having Ingress in my cluster.

1. Replaced Classic ELB with Application ELB.
2. Rules and External DNS.
3. Redirect to HTTPS.

There are few implementations of Kubernetes Ingress we can choose from and the one I'm using is [aws-alb-ingress-controller][aws-alb-ingress-controller].

## Replaced Classic ELB with Application ELB

Below is my previous setup before having Ingress.

![Cluster without Ingress](/images/cluster_without_ingress.png)

When you visited this blog, the request goes through Kubernetes 'LoadBalancer' Service
and it will be redirected to one of the Pod which will serve this lovely content :stuck_out_tongue:.
For the purpose of this simple site, this is more than sufficient.

But I'm not going to stop here since the purpose of the whole thing is to learn and we will move forward with Ingress.

Below is my current setup with Ingress.

![Cluster with Ingress](/images/cluster_with_ingress.png)

Now the previous Classic ELB has been replaced by Application ELB. ELB acts as an Ingress Proxy.
It will be the single point of entry for all external access to my cluster. The Kubernetes 'LoadBalancer' Service is
now replaced by Kubernetes 'NodePort' Services which are only accessible internally.

One benefit of having Application ELB over Classic ELB, the pricing is much cheaper :moneybag:. Yeay!.  And the goodness does not stop here.

## Rules and External DNS

Ingress Rules contains the instructions that will help Ingress Controller to redirect the request
to the correct services.

Below is an excerpt of my Ingress Rules. It says all request that matches the `host` and `path`,
should go to a particular Kubernetes 'NodePort' Service.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
  ...
spec:
  rules:
    - host: blog.taufek.dev
      http:
        paths:
         - path: /*
           backend:
             serviceName: blog-nodeport-service
             servicePort: 80
```

Another cool thing about this is the External DNS which will automatically sync with Hosted Zone in AWS Route 56.
If I need to add a new rule with new 'host', like below, this will trigger a task to update my Hosted Zone with new Record Set.
Or if I remove a particular 'host' from the rules, it will also remove the Record Set.

```
spec:
  rules:
    - host: blog.taufek.dev
      http:
        paths:
         - path: /*
           backend:
             serviceName: blog-nodeport-service
             servicePort: 80
    - host: api.taufek.dev
      http:
        paths:
         - path: /*
           backend:
             serviceName: api-nodeport-service
             servicePort: 8080
```

## Redirect to HTTPS

Not all browsers like Chrome will auto redirect my `*.dev` domain to https. I would like
to have consistent experience for all browsers so I will force all the request to HTTPS.
This is easily achieved with Ingress Rules. Below is the configuration to implement HTTPS redirect.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/certificate-arn: <ADD_YOUR_CERT_ARN_HERE>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
spec:
  rules:
    - host: blog.taufek.dev
      http:
        paths:
         - path: /*
           backend:
             serviceName: ssl-redirect
             servicePort: use-annotation
         - path: /*
           backend:
             serviceName: blog-nodeport-service
             servicePort: 80
```

## Conclusions

With this change, not only it helps me to cut down my AWS bill, it also made the cluster
to be easily managed. If I need to serve new application, I no longer need
to spawn new LoadBalancer Service, configure the SSL cert and etc. I only need to do it once for
the Application ELB that serves as Ingress Proxy. All new applications will be managed via Ingress Rules.
Although it looks more complex on the diagram but it helps if you plan to have multiple applications in your cluster.

[aws-alb-ingress-controller]: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
