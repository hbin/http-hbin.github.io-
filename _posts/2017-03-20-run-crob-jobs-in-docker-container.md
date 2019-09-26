---
layout: post
title: "Run Whenever & Cron in the Docker container"
date: 2017-03-20 23:34:16 +0800
comments: true
categories: [Tool]
tag: [Docker, Whenever, Cron]
---

Whenever is a Ruby gem that provides a clear syntax for writing and deploying cron jobs.

This blog introduces how to run your whenever cron job in the Docker container.

1. Install Cron

    Since the docker image `ruby:2.5.0` don't have a Cron builtin. We have to install it first. Open the `Dockfile` with your favorite text editer, and add the following script under the `FROM ruby:2.5.0`:

    ```
    RUN apt-get update && apt-get install -y --no-install-recommends cron \
    && rm -r /var/lib/apt/lists/*
    ```

2. Add a new task to `docker-compose.yml` file

    This result in an image named `<sample_app>_cron` after you run `docker-compose build`:

    ```
    cron:
      depends_on:
        - db
        - redis
      build: .
      command: sh bin/start-cron.sh
      volumes:
        - .:/app
    ```

3. Create a script `/the-path-to-your-app/bin/start-cron.sh` to start your cron job:

    ```
    #!/bin/sh
    bundle exec whenever --write-crontab --set environment=development
    cron -f
    ```

But after you start it with `docker-compose start` command. You might got an exception [failed to load command: rake](https://github.com/javan/whenever/issues/656). It was because of the PATH environment not set properly.

It take me a while to figure it out, but the solution is trivial, just add the following two lines at the beginning of your `config/schedule.rb` file:

    env :PATH, ENV['PATH']
    env :BUNDLE_PATH, ENV['BUNDLE_PATH']

Using `--set` to set the environment variable:

    bundle exec whenever --write-crontab --set 'environment=development&bundle_command=bundle exec'
