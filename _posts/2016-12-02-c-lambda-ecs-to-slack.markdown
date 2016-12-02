---
published: true
title: C# Lambda: ECS to slack!
layout: post
---
I created a C# AWS Lambda for fun.  I didn't like reading node.js javascript for something similar so I decided to try the new C# Lambda.  Hopefully, this is a good sample for others.

Code is all here: [Visibility.Lambda.Slack repo](https://github.com/Visibilityltd/Visibility.Lambda.Slack)

## Background ##

* [Lambda C# Progamming Model](http://docs.aws.amazon.com/lambda/latest/dg/dotnet-programming-model.html)
* [Monitor Cluster State with Amazon ECS Event Stream](https://aws.amazon.com/blogs/compute/monitor-cluster-state-with-amazon-ecs-event-stream/)
* I'm reusing build steps from here: [.NET Core 1.1 building with Docker and Cake](https://adamhathcock.github.io/2016/11/22/net-core-1-1-building-with-docker-and-cake.html)

## Building ##

I have to use Docker to build on OS X.  There is probably something I'm missing.

`./docker.sh` does the following:

* Uses a build container to build for debian using Cake
* copies the publish directory out of the container to `publish/`
* zips the `publish/` directory to a zip file ready to be uploaded

## Installing into Lambda ##

* The base instructions pretty much work
* Handler name: `Visibility.Lambda.Slack::Visibility.Lambda.Slack.LambdaHandler::EcsCloudWatch`
* Use an environment variable for your slack hook url called: `slack_webhook_url`

## Fun Notes ##

* All of the Lambda code provided is on .NET Core 1.0.  Lots of conflicts happen trying to upgrade to .NET Core 1.1
* `CloudWatchEvent<>` is a base POCO for CloudWatch Event logs
* `EcsEventDetail` is the detail POCO for ECS data.
* Everything is ugly and early quality.  I hope to build this out more with more detail for ECS and other AWS events.