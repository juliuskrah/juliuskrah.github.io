---
layout:     post
title:      Build a Laravel Docker Image Using a Dockerfile
date:       2020-10-01 10:27:31 +0000
categories: blog
tags:       php laravel docker
section:    blog
author:     juliuskrah
---

> Laravel is a modern web application framework for artisans.

# Introduction

Building a dockerfile for a `Laravel` application does not need to be tedious. The good folks at
[The Coding Machine][thecodingmachine]{:target="_blank"} have created several base images that we can leverage.

## Prerequisites

You need to have the following setup on your development machine to proceed with this tutorial

1. [PHP >= 7.3][PHP]{:target="_blank"}
2. [Composer][Composer]{:target="_blank"}
3. [Laravel][Laravel]{:target="_blank"}
4. [Docker][Docker]{:target="_blank"}

## Let's get started

Create a new laravel project

```bash
> laravel new laravel-app
```

and navigate into the `laravel-app` directory and execute the following command:

```bash
> php artisan serve
```

Try it out in your browser [localhost:8000](http://localhost:8000){:target="_blank"}

Before we create our `Dockerfile` lets first create a `.dockerignore` file:

file: `laravel-app\.dockerignore`

```
node_modules/
public/hot/
public/storage/
storage/*.key
vendor/
.env.backup
.phpunit.result.cache
Homestead.json
Homestead.yaml
npm-debug.log
yarn-error.log
Dockerfile
docker-compose.yml
docker-compose.yaml
**.git
.idea
**.gitignore
**.gitattributes
**.sass-cache
```

We can now create our `Dockerfile`:

file: `laravel-app\Dockerfile`

```dockerfile
ARG PHP_EXTENSIONS="intl pdo_mysql pdo_pgsql bcmath"

FROM thecodingmachine/php:7.4-v3-slim-apache as php_base
LABEL maintainer "Julius Krah <juliuskrah@gmail.com>"

ENV TEMPLATE_PHP_INI=production
COPY --chown=docker:docker . /var/www/html
RUN composer install --quiet --optimize-autoloader --no-dev

FROM node:10 as node_dependencies
WORKDIR /var/www/html

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=false
COPY --from=php_base /var/www/html /var/www/html

RUN npm set progress=false && \
  npm config set depth 0 && \
  npm install && \
  npm run prod && \
  rm -rf node_modules

FROM php_base
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
COPY --from=node_dependencies --chown=docker:docker /var/www/html /var/www/html
```

Build and tag the image:

```bash
> docker build -t laravel-app .
```

Time to test our image with a container

```bash
> docker run --rm --name laravel -e DB_CONNECTION=pgsql -p 8000:80 laravel-app
```

NOTE: *You can replace all options in the `.env` file with environment variables on the command line e.g.
`-e DB_CONNECTION=pgsql`*

## Conclusion

An optional step is to tag and upload your image to docker hub:

```bash
> docker tag laravel-app <username>/laravel-app
> docker login
> docker push <username>/laravel-app
```

In this post we discussed creating a laravel project and making an OCI image out of it. Until the next post, keep
doing cool things :+1:.

[Composer]:                       https://getcomposer.org/download/
[Docker]:                         https://www.docker.com/products/docker-desktop
[Laravel]:                        https://laravel.com/docs/8.x#installation
[PHP]:                            https://www.php.net/
[thecodingmachine]:               https://github.com/thecodingmachine/docker-images-php