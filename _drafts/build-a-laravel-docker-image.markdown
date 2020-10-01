---
layout:     post
title:      Build a Laravel Docker Image Using a Dockerfile
categories: blog
tags:       php laravel docker
section:    blog
author:     juliuskrah
repo:       jasper-html-mail/tree/jasper-rest-multi
---

> as Stuff

# Introduction

-- Generate a laravel project

## Prerequisites

1. PHP
2. Composer
3. Laravel

## Create a dockerfile

```dockerfile
ARG PHP_EXTENSIONS="apcu bcmath imagick gd pgsql pdo_pgsql"

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

## Test

Building the image

```bash
> docker build -t laravel-demo .
```

Running the image

```bash
> docker run --rm --name laravel -e DB_CONNECTION=pgsql -p 8000:80 laravel-demo
```
