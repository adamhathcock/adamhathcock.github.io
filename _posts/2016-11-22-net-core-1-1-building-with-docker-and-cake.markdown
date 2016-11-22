---
published: false
title:  .NET Core 1.1 building with Docker and Cake
layout: post
tags: [dotnetcore, cake, docker]
---
# .NET Core 1.1 building with Docker and Cake

I'm going to attempt to catalog how I'm using Docker to test and build containers that are for deployment into Amazon ECS.

## Build Process
1. Use `Dockerfile.build`
    - Uses Cake:
        1. `dotnet restore`
        2. `dotnet build`
        3. `dotnet test`
        4. `dotnet publish`
2. Save running image to container
3. Copy `publish` directory out of container
4. Use `Dockerfile`
    - Copy `publish` directory into image
5. Push built image to ECS


## Driving the build: Cake

I love [Cake](http://cakebuild.net/) and have contributed some minor things to it.  It does support .NET Core.  However, the `nuget.exe` used to drive some critical things like `nuget push` does not.  `push` is actually the only command I need that isn't on .NET Core.  So I standardized on requiring Mono for just the build container.

My base Cake file: `build.cake`
```
var target = Argument("target", "Default");
var tag = Argument("tag", "cake");

Task("Restore")
  .Does(() =>
{
    DotNetCoreRestore("src/\" \"test/\" \"integrate/");
});

Task("Build")
    .IsDependentOn("Restore")
  .Does(() =>
{
    DotNetCoreBuild("src/**/project.json\" \"test/**/project.json\" \"integrate/**/project.json");
});

Task("Test")
    .IsDependentOn("Build")
  .Does(() =>
{
    var files = GetFiles("test/**/project.json");
    foreach(var file in files)
    {
        DotNetCoreTest(file.ToString());
    }
});

Task("Publish")
    .IsDependentOn("Test")
  .Does(() =>
{
    var settings = new DotNetCorePublishSettings
    {
        Framework = "netcoreapp1.1",
        Configuration = "Release",
        OutputDirectory = "./publish/",
        VersionSuffix = tag
    };
                
    DotNetCorePublish("src/Server", settings);
});

Task("Default")
    .IsDependentOn("Test");

RunTarget(target);
```
I broke out all the steps as I often run Cake for each step during development.  You'll notice that each `dotnet` command behaves differently.  It's very annoying.

I have a project structure that usually goes like this:
- `src` - Source files
- `test` - Unit tests for those source files
- `integrate` - Integration tests that should run separately from unit tests.
- `misc` - Other code stuff

Other things to notice:
- `Default` is test.  Don't want to accidently publish
- `publish` has a hard-coded entry point.  Probably should make that argument.
- `tag` is a tag I want to tag the published build with.  I want to see something unique for each publish.  I default this with `cake` for local publishes.

## The Build Container: `Dockerfile.build`

I actually started with following the little HOW-TO from the ASP.NET team from [here:](https://github.com/aspnet/aspnet-docker/tree/master/1.1.0/jessie/build-projectjson)

```
FROM cl0sey/dotnet-mono-docker:1.1-sdk

ARG TAG=docker
ENV TAG ${TAG}

WORKDIR /app
RUN mkdir /publish

COPY . .
RUN ./build.sh -t publish --scriptargs "--tag=${TAG}"
```

Notice the source image: [cl0sey/dotnet-mono-docker:1.1-sdk](https://hub.docker.com/r/cl0sey/dotnet-mono-docker/)

Someone was nice enough to already make a Docker image with Mono on top of the base `microsoft/dotnet:1.1-sdk-projectjson` image.  The SDK image is what is needed for using all of the `dotnet cli` commands that aren't just running.

Notice:
- `ARG` and `ENV` declarations for specifying the `tag` variable.  I think `ARG` declares it and `ENV` allows it to be used as a `bash`-like variable.
- creating a `publish` directory.
- How I pass the `tag` variable to the `Cake` script.

## The Deployment Container: `Dockerfile`

```
FROM microsoft/dotnet:1.1.0-runtime

COPY ./publish /app
WORKDIR /app

EXPOSE 5000

ENV ASPNETCORE_ENVIRONMENT beta

ENTRYPOINT ["dotnet", "Server.dll"]
```

Notice:
- I actually use the official runtime image.
- `COPY` command to grab the local `publish` directory and put it in the `app` directory inside the container.
- I keep the default 5000 port.  Why not?  It's all hidden in AWS.
- I just declared my environment to be `beta` instead of `staging`
- `ENTRYPOINT` has to be an array of strings.  `Server.dll` is the executable assembly.

## Hanging It All Together: CircleCI

I'm using [CircleCI](https://circleci.com/) as my CI service because it's free/cheap.  Also, it runs Docker and can do Docker inside Docker.  The `docker` commands will work just about anywhere though.

```
machine:
  services:
    - docker

dependencies:
  override:
    - docker info

test:
  override:
    - docker build -t build-image --build-arg TAG="${CIRCLE_BRANCH}-${CIRCLE_BUILD_NUM}" -f Dockerfile.build .
    - docker create --name build-cont build-image


deployment:
  beta:
    branch: master
    commands:
    - docker cp build-cont:/app/publish/. publish/
    - docker build -t server-api:latest .
    - docker tag server-api:latest $AWS_ACCOUNT_ID.dkr.ecr.eu-west-1.amazonaws.com/server-api:$CIRCLE_BUILD_NUM
    - ./push.sh
```
Notice:
- `test` phase
    - The `test` phase does `docker build` on `Dockerfile.build`  This file does everything, including publish. The image is tagged as `build-image`.
    - `test` phase also creates a container called `build-cont` for possible deployment.
    - My `tag` is made of the branch name plus the build number.  These are `CircleCI` variables.
- `deployment` phase
    - named `beta` I could have more environments for deployment, I guess.
    - locked to the `master` branch.  When I push feature branches, only the `test` phase runs to test things.  Only when merged into `master` does it deploy.
    - `docker cp` copies the `publish` directory out of the `build-cont` container.
    - `Dockerfile` is used with `docker build` and tagged as `server-api:latest`
    - I also explicitly tag the image with my AWS ECS specific name.  `CircleCI` hides my AWS account id in an environment variable for me.
    - `push.sh` actually does the push to AWS.

## push.sh to AWS ECS 
Finally, I want to save my Docker image.

```
#!/usr/bin/env bash

configure_aws_cli(){
	aws --version
	aws configure set default.region eu-west-1
	aws configure set default.output json
}

push_ecr_image(){
    eval $(aws ecr get-login --region eu-west-1)
    docker push $AWS_ACCOUNT_ID.dkr.ecr.eu-west-1.amazonaws.com/visibility-api:$CIRCLE_BUILD_NUM
}

configure_aws_cli
push_ecr_image
```

The bash script is copied in part from something else more complicated.  You can't just do the push command from the `circle.yaml` because of the need to use `eval` to login to AWS.  My AWS push creds are also locked in a CircleCI environment variable that the `aws ecr get-login` command expects.
