---
layout: post
title:  "Deploy Jekyll Site to Heroku"
date:   2020-06-14 15:30:00 +0800
categories: kubernetes
hero_src: jekyll_heroku.png 
---

Generally there are 2 ways to deploy Jekyll static site to Heroku.
1. Deploy as rails app using web server gem such as `puma`, `passenger`, `thin`, 'etc'.
2. Deploy as Docker image running `nginx` service.

In this post, I chose 2nd approach since I like to use Docker image.

## Prepare Dockerfile

We will serve this Jekyll site via nginx service. In order to map Heroku port
number to our nginx service, we will need to use below `default.conf` template.

```nginx
server {
  # Placeholder for Heroku port
  listen  $PORT;

  location / {
    root  /usr/share/nginx/html;
    index index.html;
  }
}
```

Dockerfile.

```dockerfile
FROM nginx:alpine

# copy nginx `default.conf`
COPY nginx/default.conf /etc/nginx/conf.d/
# copy Jekyll generated files
COPY _site /usr/share/nginx/html/

# replace $PORT placeholder with HEROKU given port in default.conf and run nginx service
CMD sed -i -e 's/$PORT/'"$PORT"'/g' /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'
```

You can named this file as `Dockerfile` or have a suffix to indiciate the
Heroku process. For instance, if you want to run this application as `web`
process in Heroku, you could name it as `Dockerfile.web`. This works well if
you have multiple `Dockerfile`s with different process such as `web`, `worker`, or
`clock`. Later on, when you run `heroku container:push --recursive`, it will
build all the `Dockerfile`s with the correct suffix and push them to Heroku docker
registry.

## Configure Github Action

Even without Github Action, all I need is Heroku CLI to deploy this Docker
image using below commands.

```bash
> bundle exec jekyll build

> heroku container:login

> heroku container:push -a <HEROKU_APP_NAME> --resursive

> heroku container:release -a <HEROKU_APP_NAME> web
```

Basically, in Github Actions, we are running above commands. Below is the full
Github Action workflow yaml.

{% highlight yaml %}
{% raw %}
name: Push Container to Heroku

on:
  push:
    branches:
      - master

jobs:
  build:
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

    - name: Login to Heroku Container registry
      env:
        # run `heroku authorizations:create` to generate the HEROKU_API_KEY
        HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
      run: heroku container:login

    - name: Build and push
      env:
        HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
      run: heroku container:push -a ${{ secrets.HEROKU_APP_NAME }} --recursive

    - name: Release
      env:
        HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
      run: heroku container:release -a ${{ secrets.HEROKU_APP_NAME }} web
{% endraw %}
{% endhighlight %}

With a push to `master` branch, it will trigger a build which will
1. Checkout code from `master` branch.
2. Generate Jekyll static files.
3. Build and push the Docker image to heroku registry.
4. Deploy the Docker image to Heroku with `web` process.

## Conclusion

This approach not only work for static site, but also for other types
of application such as Rails application. Having a Dockerfile will give you
full control on what other linux packages you wish to install within the
application server instance such as ImageMagick. It will also give you
consistent environment between running your application on your local machine
and on Heroku.
