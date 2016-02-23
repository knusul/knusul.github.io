---
layout: post
title: ""Logging with Docker""
date: 2016-02-22 16:59:05 +0100
comments: true
categories: docker
---

There is great article how to configure centralized logging for docker with cloudwatch:
```
https://blogs.aws.amazon.com/application-management/post/TxFRDMTMILAA8X/Send-ECS-Container-Logs-to-CloudWatch-Logs-for-Centralized-Monitoring
```
It assumes that the application can send logs to other container running rsyslog.
We will extend the example to work with locally saved logs.
In order to make it work we need to:

1) Create one container that will configure cloudwatch and rsyslog that listen to the events.

2) Configure rsyslog on second devise that will listen to file changes in logs folder and will send logs to second rsyslog service running on another machine

I decided to create another rsyslog agent running together with my application because I'm reusing the docker cloudwatch container.

First lets remove /dev/xconsole from rsyslog config. It breaks the rsyslog because most docker images does not have dev/xconsole.
Open aws cloudwatch ecs project and edit Dockerfile
```
RUN rm /etc/rsyslog.d/50-default.conf
```
Next we need to configure rsyslog on our app machine. We need rsyslog version > 8.15.0 to use wildcard imfile as a log source(see rsyslog.conf):
```
RUN apt-get update
RUN apt-get -y install libestr-dev  liblogging-stdlog-dev  libgcrypt-dev uuid-dev libjson0-dev  libz-dev
RUN wget http://www.rsyslog.com/files/download/rsyslog/rsyslog-8.16.0.tar.gz
RUN tar xvzf rsyslog-8.16.0.tar.gz
RUN cd rsyslog-8.16.0 && ./configure --enable-imfile && make && make install
COPY rsyslog.conf /etc/rsyslog.conf
RUN pip install supervisor
COPY supervisord.conf /usr/local/etc/supervisord.conf
CMD ["/usr/local/bin/supervisord"]
```

supervisord.conf is pretty simply:

```
[supervisord]
nodaemon=true

[program:rsyslogd]
command=/usr/local/sbin/rsyslogd -n

[program:awslogs]
command=your-app-with-logs

```

rsyslog.conf sends logs written to file to rsyslog on machine running cloudwatch agent
```
module(load="imfile")
input(type="imfile" file="/usr/src/app/logs/location/*.log" tag="spider_locations" facility="local6")
```

