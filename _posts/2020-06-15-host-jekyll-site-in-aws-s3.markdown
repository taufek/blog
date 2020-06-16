---
layout: post
title:  "Host Jekyll Site in AWS S3"
date:   2020-06-15 13:40:00 +0800
categories: kubernetes
hero_src: jekyll_s3.png
---

In previous [post]({% post_url 2020-06-14-deploy-jekyll-site-to-heroku %}),
I've wrote about on how to push a Jekyll site to Heroku. In this post it will
be about how I host this blog in AWS S3 bucket.

Turns out it is much simpler to do it in S3.

## Create a S3 bucket

Create a S3 bucket and enable Web Hosting in the properties.

## Configure Github Action 

You will need to configure following secrets in your Github setting.
1. `AWS_S3_BUCKET` with your bucket name.
2. `AWS_ACCESS_KEY_ID` with your AWS access key id.
2. `AWS_SECRET_ACCESS_KEY` with your AWS secret access id.

{% highlight yaml %}
{% raw %}
name: Sync to S3

on:
push:
branches:
    - master

        jobs:
        deploy:
        runs-on: ubuntu-latest
        steps:
    - uses: actions/checkout@v1

    - name: Retrieve Gem Bundles Cache
        uses: actions/cache@v2
        with:
        path: _bundle
        key: ${{ runner.os }}-gems-${{ hashFiles('**/Gemfile.lock') }}
        restore-keys: |
        ${{ runner.os }}-gems-

    - name: Generate site
        run: |
        mkdir -p _bundle/cache
        chmod a+rwx -R .
        docker-compose run -e JEKYLL_ENV=production jekyll bundle install
        docker-compose run -e JEKYLL_ENV=production jekyll bundle exec jekyll build --trace

    - uses: jakejarvis/s3-sync-action@master
        with:
        args: --acl public-read --follow-symlinks --delete
        env:
        AWS_S3_BUCKET: ${{ secrets.AWS_S3_BUCKET }}
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_REGION: 'ap-southeast-1'
        SOURCE_DIR: '_site'
{% endraw %}
{% endhighlight %}

With a push to `master` branch, it will trigger a build which will
1. Checkout code from `master` branch.
2. Generate Jekyll static files.
3. Push generated files inside `_site` folder to your S3 bucket.

## Conclusion

This seems like a cheaper option for hosting a static site and have SSL
configured in CloudFront compared to paying for a dyno in Heroku to host a site
with SSL. For Heroku, I will need to pay atleast $7/month for a dyno and for
CloudFront it will based on usage. Since my blog does not have much visitors,
my AWS bill is very small which is around $1/month. This includes S3, Route 53
and CloudFront services.

