---
layout: post
title:  "Let Cert-Manager Manages LetsEncrypt Certificates"
date:   2020-06-09 00:20:00 +0800
categories: kubernetes
hero_src: rusty-padlock.jpg
---

Since this blog is using `*.dev` domain, I will need to configure SSL
certificate in order to serve this content in HTTPS. I'm using [LetsEncrypt] to
issue my certificate and the certificate only valid for 3 months. I will need
to renew them periodically. I can either do that manually when the certificate
is almost expire or I can automate the process.

In this post, we will look into how to automate the certificate renewal with [cert-manager].

I've been using [aws-alb-ingress-controller] as Ingress implementation in my
cluster and but for this, I will need to swap to another Ingress implementation
which is [nginx-ingress]. This is because currently, there is no way for the
[cert-manager] to upload the certificate to the ALB.

## Pre-requisite

Add below IAM policy to your "node" role.

```
"Version": "2012-10-17",
"Statement": [
  {
    "Effect": "Allow",
    "Action": "route53:GetChange",
    "Resource": "arn:aws:route53:::change/*"
  },
  {
    "Effect": "Allow",
    "Action": [
      "route53:ChangeResourceRecordSets",
      "route53:ListResourceRecordSets"
    ],
    "Resource": "arn:aws:route53:::hostedzone/*"
  },
  {
    "Effect": "Allow",
    "Action": "route53:ListHostedZonesByName",
    "Resource": "*"
  }
]
```

## Install Cert Manager

Run below to install [cert-manager] and its dependencies.

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.yaml
```

Apply below `Issuer`:

{% highlight yaml %}
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
  server: https://acme-v02.api.letsencrypt.org/directory
  email: taufek@gmail.com

  privateKeySecretRef:
    name: letsencrypt-prod

  solvers:
      - selector:
      dnsNames:
          - blog.taufek.dev
          http01:
            ingress:
              class: nginx
{% endhighlight %}

In this `Issuer`, we configure to use:
1. [LetsEncrypt] production api.
2. `http01` challenge to verify that we own the domain. The other method is `dns01`.
3. `nginx` ingress implementation.

And lastly, we apply below `Ingress` rule:

{% highlight yaml %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    cert-manager.io/issuer: 'letsencrypt-prod'
  spec:
    tls:
      - hosts:
        - blog.taufek.dev
        secretName: blog-taufek-dev-tls
    rules:
      - host: blog.taufek.dev
          http:
          paths:
           - path: /
               backend:
                 serviceName: blog-nodeport-service
                 servicePort: 80
{% endhighlight %}

Wait for a minute for [cert-manager] to do its thing. It will:
1. Send a CertificationRequest to [LetsEncrypt] API.
2. Setup and API to fulfill [LetsEncrypt] challenge.
3. Generate the certificate files.
4. Apply the new/renewed certificate to [nginx-ingress].

You can run below to check the certificate status

```
> kubectl describe certificate blog-taufek-dev

Name:         blog-taufek-dev-tls
...
API Version:  cert-manager.io/v1alpha3
Kind:         Certificate
...
Events:
Type    Reason     Age    From          Message
----    ------     ----   ----          -------
Normal  Requested  5m20s  cert-manager  Created new CertificateRequest resource "blog-taufek-dev-tls-918422041"
Normal  Issued     4m44s  cert-manager  Certificate issued successfully
```

And that's it. I have successfully setup [LetsEncrypt] certificate for my blog it it will be auto renew by [cert-manager] when the time comes.

## Bonus

[cert-manager] also comes with [kubectl-plugin]. When installed, there are few additional commands I can use via `kubectl` related to certificate management.

For example, I can run below to renew all my certificates
```
kubectl cert-manager renew --all
```

## Conclusions

Certificate renewal is a must for production grade application. You wouldn't
want to miss renewing your applications certificate when it almost expires. The
only thing that made me not to use this implementation is because it requires
Classic ELB and I'm still evaluating if the cost is still within my budget. If
I were to setup for a profitable application, this is definitely my choice of
implementation.

[aws-alb-ingress-controller]: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
[nginx-ingress]: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
[cert-manager]: https://cert-manager.io/docs/
[LetsEncrypt]: https://letsencrypt.org/
[kubectl-plugin]: https://cert-manager.io/docs/usage/kubectl-plugin/
