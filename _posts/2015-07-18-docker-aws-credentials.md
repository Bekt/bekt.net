---
layout: post
title: Sharing AWS Credentials Between Users On a Docker Container
permalink: /p/docker-aws-credentials
tags: [docker, aws]
---

**TL;DR**:
AWS CLI configurations and credentials are stored in the user's home directory by
default (`~/.aws`). This becomes a problem when other users such as
`root`, `www-data`, `nobody`, or cron jobs need access to these credentials.
This post shows how to get around this in a Docker environment.

[Docker](https://www.docker.com/) is a great application containment tool for
building and shipping software of any type. You can learn more about Docker
[here](https://www.docker.com/whatisdocker). We love and use Docker at
[Humanlink](https://www.humanlink.co) with
[AWS Elastic Beanstalk](http://aws.amazon.com/elasticbeanstalk/).

Being on Elastic Beanstalk means utilizing other amazing Amazon services as well.
[AWS CLI](http://aws.amazon.com/cli/) and [boto](https://github.com/boto/boto)
(for Python developers) are the de-facto tools for interacting with AWS.

There are a few ways to provide credentials to these tools.
On production, it is a good idea to serve them as environment
variables (such as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`).
During development, however, it is recommended that these values are set via
the `aws configure` command, which in turn places config files under `~/.aws/`.

I was planning on having a full-blown example with detailed explanations,
but that quickly became a long post and lost its focus.

In short, we do not want to pollute the Dockerfile just for the development environment. When running the Docker image locally, we can mount the `~/.aws` directory AND set the `$HOME` environment variable. Otherwise, `$HOME` defaults
to `/root` which causes permission problems for non-root users.

Since mounted file permissions are copied over to the Docker container, we
need to give read permissions to everyone (on your host machine).

{% capture c0 %}$ chmod 644 ~/.aws/*
{% endcapture %}

<pre><code class="bash">{{ c0 | escape }}</code></pre>

We can now run the container:

{% capture c1 %}$ docker run -it --rm \
  -e "HOME=/home" \
  -v $HOME/.aws:/home/.aws \
  myapp
{% endcapture %}

<pre><code class="bash">{{ c1 | escape }}</code></pre>

Now the credentials are available system-wide in the Docker container for any
user to use and communicate with AWS.
