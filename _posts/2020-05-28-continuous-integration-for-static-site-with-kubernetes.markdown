---
layout: post
title:  "Continuous Integration for Static Site with Kubernetes"
date:   2020-05-28 00:00:16 +0800
categories: kubernetes
hero_src: domore.jpg
---

# Problem Statement
I always wanted to work with Kubernetes but the subject comes with steep
learning curve. Especially for someone who does not work directly with software
infrastructure on day-to-day basis. The best way to learn is to actually use it
in actual setting. This is my first attempt to use Kubernetes to serve a
content on the Internet.

# What: Non-Fictitious CI/CD with Kubernetes
My plan is to setup a working and simple Continuous Integration and Continuous
Deployment (CI/CD) workflow that integrates with Kubernetes cluster. What can
be simpler than deploying a static site. I might as well I start a new blog
site. This blog site is the result of CI workflow that I'll be describing
below.

# How: Services that Build and Deploy This Blog to Kubernetes Cluster
Services involved in this CI:
1. GitHub Action. Workflow orchestration tool that executes steps in CI.
2. Github Package Registry. Docker image registry provided by Github.
3. Kubernetes cluster. At the point of writing this post I'm running my cluster on AWS with spot instances provisioned by [kops][kops-repo]. Still evaluating if this is the cheapest method for running Kubernetes cluster. (This is for future blog post)

Below are the high level steps:
1. Make changes to my blog repository.
2. Push to origin with tag (i.e `v1.0.0`). Pushing to master without tag will not trigger the build which allow me to push my changes to master multiple times without triggering the deployment step.
3. GitHub Action will do its magic to checkout, build and push newly created image to Kubernetes cluster.

Below is the GitHub Action yml file which wires everthing together.
{% highlight yaml %}
{% raw %}
name: Docker

on:
  push:
    tags:
      - v*
env:
  IMAGE_NAME: blog

jobs:
  push:
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    steps:
      - uses: actions/checkout@v2

      - name: Retrieve Gem Bundles Cache
        uses: actions/cache@v1
        with:
          path: _bundle
          key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile.lock') }}
          restore-keys: |
            ${{ runner.os }}-gems-

      - name: Generate site
        run: |
          mkdir -p _bundle/cache
          chmod a+rwx -R .
          docker-compose run jekyll bundle install
          docker-compose run jekyll bundle exec jekyll build --trace

      - name: Build image
        run: docker build . --file Dockerfile --tag $IMAGE_NAME

      - name: Check bundle
        run: |
          ls _bundle/gems

      - name: Log into registry
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login docker.pkg.github.com -u ${{ github.actor }} --password-stdin

      - name: Push image
        run: |
          IMAGE_ID=docker.pkg.github.com/${{ github.repository }}/$IMAGE_NAME

          # Change all uppercase to lowercase
          IMAGE_ID=$(echo $IMAGE_ID | tr '[A-Z]' '[a-z]')

          # Strip git ref prefix from version
          VERSION=$(echo "${{ github.ref }}" | sed -e 's,.*/\(.*\),\1,')

          # Strip "v" prefix from tag name
          [[ "${{ github.ref }}" == "refs/tags/"* ]] && VERSION=$(echo $VERSION | sed -e 's/^v//')

          # Use Docker `latest` tag convention
          [ "$VERSION" == "master" ] && VERSION=latest

          echo IMAGE_ID=$IMAGE_ID
          echo VERSION=$VERSION

          docker tag $IMAGE_NAME $IMAGE_ID:$VERSION
          docker push $IMAGE_ID:$VERSION

          echo "::set-env name=VERSION::$VERSION"

      - name: Check Version
        id: check_version
        run: |
          echo ::set-output name=version::$VERSION

      - name: Update k8s deployment
        uses: Consensys/kubernetes-action@master
        env:
          KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
        with:
          args: set image deployment/blog-deployment blog=docker.pkg.github.com/taufek/blog/blog:${{ steps.check_version.outputs.version }}
{% endraw %}
{% endhighlight %}

# Github Action Yaml Breakdown

Trigger this Github Action workflow if the push to origin includes tag starting with `v`. For example `v1.0.0`.

{% highlight yaml %}
on:
  push:
    tags:
      - v*
{% endhighlight %}

Standard steps to checkout and retrieve cache (if there is any).

{% highlight yaml %}
{% raw %}
  - uses: actions/checkout@v2

  - name: Retrieve Gem Bundles Cache
    uses: actions/cache@v1
    with:
      path: _bundle
      key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile.lock') }}
      restore-keys: |
        ${{ runner.os }}-gems-
{% endraw %}
{% endhighlight %}


Generate the static site via Jekyll command. I use Docker image to skip Jekyll setup steps.

{% highlight yaml %}
  - name: Generate site
    run: |
      mkdir -p _bundle/cache
      chmod a+rwx -R .
      docker-compose run jekyll bundle install
      docker-compose run jekyll bundle exec jekyll build --trace
{% endhighlight %}

Build new Docker image for this blog. `$IMAGE_NAME` value is `blog`.

{% highlight yaml %}
{% raw %}
  - name: Build image
    run: docker build . --file Dockerfile --tag $IMAGE_NAME
{% endraw %}
{% endhighlight %}

Login to Github Package Registry. This is required before pushing the image to the registry.

{% highlight yaml %}
{% raw %}
  - name: Log into registry
    run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login docker.pkg.github.com -u ${{ github.actor }} --password-stdin
{% endraw %}
{% endhighlight %}

Push the Docker image registry as `docker.pkg.github.com/taufek/blog/blog:1.0.0`.

{% highlight yaml %}
{% raw %}
  - name: Push image
    run: |
      IMAGE_ID=docker.pkg.github.com/${{ github.repository }}/$IMAGE_NAME

      # Change all uppercase to lowercase
      IMAGE_ID=$(echo $IMAGE_ID | tr '[A-Z]' '[a-z]')

      # Strip git ref prefix from version
      VERSION=$(echo "${{ github.ref }}" | sed -e 's,.*/\(.*\),\1,')

      # Strip "v" prefix from tag name
      [[ "${{ github.ref }}" == "refs/tags/"* ]] && VERSION=$(echo $VERSION | sed -e 's/^v//')

      # Use Docker `latest` tag convention
      [ "$VERSION" == "master" ] && VERSION=latest

      echo IMAGE_ID=$IMAGE_ID
      echo VERSION=$VERSION

      docker tag $IMAGE_NAME $IMAGE_ID:$VERSION
      docker push $IMAGE_ID:$VERSION

      echo "::set-env name=VERSION::$VERSION"

  - name: Check Version
    id: check_version
    run: |
      echo ::set-output name=version::$VERSION
{% endraw %}
{% endhighlight %}

Update Kubernetes Deployment with new image tag.

{% highlight yaml %}
{% raw %}
  - name: Update k8s deployment
    uses: Consensys/kubernetes-action@master
    env:
      KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
    with:
      args: set image deployment/blog-deployment blog=docker.pkg.github.com/taufek/blog/blog:${{ steps.check_version.outputs.version }}
{% endraw %}
{% endhighlight %}

You can view the blog repository here [taufek/blog][taufek-blog].

I'm planning to post more Kubernetes content in the future as I learn more about Kubernetes.

Keep watching this space.

[taufek-blog]: https://github.com/taufek/blog
[kops-repo]: https://github.com/kubernetes/kops
