---
layout: post
title: "AWS Lambda. The bad parts"
date: 2016-01-05 15:31:55 +0000
comments: true
categories: 
---


Amazon about their Lambda service: "AWS Lambda is a zero-administration compute platform for back-end web developers that runs your code for you in the AWS cloud and provides you with a fine-grained pricing structure"
I spend some time with a microservice-awslambda architecture. Here I share numerous problems I encountered:
1) Testing/Staging env requires seperate lambda function. We can not deploy same lambda function with different version to different APIs.
2) The Lambda environment is archaic: Node v0.9 without ECMA6 support and Python 2.7. 
3) Difficult to deploy Python with shared libraries. When deploying python function with precompiled shared library we have to build it on same architecture like AWS Lambda, otherwise it won't work. We can not simply deploy liblxml build on MacOSX. In order to compile linked libraries it's necessery to build them on ec2 instance and then deploy to Lambda. Seems like too much work for simple task.

# Conclusion
There are great benefits in an architecture build on Lambda functions. Unfortunately it comes in a costs. 
It might work well for very simple services but when we need to create anything more complicated than 'hello world' its wise to move to other solution. Maybe Dockeris a thing to go. If you want to share your thoughts about Lambda AWS architecture please comment.

