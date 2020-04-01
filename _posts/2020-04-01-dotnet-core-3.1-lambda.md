---
layout: post
title:  ".NET Core 3.1 fixes cold starts on AWS Lambda"
date:   2020-04-01 00:00:00 -0500
categories: programming
---

As of [yesterday](https://aws.amazon.com/blogs/compute/announcing-aws-lambda-supports-for-net-core-3-1/), AWS announced support for the .NET Core 3.1 runtime on AWS Lambda. This has been an [eagerly anticipated](https://github.com/aws/aws-lambda-dotnet/issues/554) upgrade as there is reason to believe that the long-standing and [well documented](https://mikhail.io/serverless/coldstarts/aws/) issues with cold starts should be somewhat mitigated.

This optimism comes from the introduction of the `ReadyToRun` [compilation option](https://docs.microsoft.com/en-us/dotnet/core/whats-new/dotnet-core-3-0#readytorun-images) in .NET Core 3.0 which allows your code to be pre-compiled for a specific target runtime, reducing the amount of just-in-time (JIT) compilation required on program startup. The trade-off is that the binary will be larger in size and not cross-platform compatible, but for .NET Lambda deployables specifically I think most will agree this is an acceptable one to make. In addition, the new performanced-focused `System.Text.Json` library introduced in .NET 3.1 should also help somewhat with initialization by making it quicker to deserialize that Lambda event payload, though I imagine the impacts from this may vary quite a bit depending on the use case.

I ran some quick before/after experiments using a [dummy Application Load Balancer function](https://github.com/vp89/lambda21test) I already had that targeted .NET Core 2.1. I upgraded this Lambda using the instructions in the [announcement blog post](https://aws.amazon.com/blogs/compute/announcing-aws-lambda-supports-for-net-core-3-1/) by simply having it target 3.1 and using the new `Amazon.Lambda.Serialization.SystemTextJson` instead of the old one based on Newtonsoft.Json. I then packaged it using `ReadyToRun` by using a Docker image with the 3.1 SDK on it (instructions below), uploaded the new .zip to Lambda and changed the runtime to 3.1. Everything worked immediately with no hiccups on an existing function which was an added bonus.

If you look at the code, you may think that this function doesn't really *do* anything, well that's kind of the point. We are only interested in the base overhead that the .NET Core Lambda runtime provides, so we can evaluate it's feasibility for specific use cases. One thing I'm not really showing here is the impact of ReadyToRun on a Lambda that pulls in a lot of depedencies, I will leave that for another future round of testing as I wanted to get a quick "first-take" with this just being announced yesterday.

### Results

#### Old 2.1 ALB function

Memory|Test Run #1 (ms)|Test Run #2 (ms)|Test Run #3 (ms)
---|---|---|---
128mb|3214.66|3266.76|3204.84
256mb|1416.55|1564.59|1479.23
512mb|704.10|712.95|736.18
1024mb|355.97|356.11|352.68
2048mb|169.25|205.09|204.44

#### New 3.1 ALB function (Linux ReadyToRun + System.Text.Json)

Memory|Test Run #1 (ms)|Test Run #2 (ms)|Test Run #3 (ms)
---|---|---|---
128mb|881.83|905.15|929.12
256mb|422.52|431.35|440.88
512mb|187.82|195.23|192.24
1024mb|97.71|96.34|93.74
2048mb|64.10|62.64|59.46

### Verdict

These are obviously really impressive improvements. While it is still relatively "unusable" at 128-256mb for latency sensitive workloads, the overhead from 512mb and above is quite acceptable and does now make this platform (and the entire serverless paradigm) much more attractive in my opinion.

### Preparing a Linux optimized Lambda using ReadyToRun and Docker

First create this simple Dockerfile with the 3.1 SDK on it:

```
FROM mcr.microsoft.com/dotnet/core/sdk:3.1
COPY . /app
WORKDIR /app
```

Now create an image, then start a container and run a shell on it:
```
docker build --tag <imageName> .
docker run --entrypoint "/bin/sh" -it <imageName>
```

First we'll need to install a few pre-requisites:
```
apt update
apt install zip
dotnet tool install -g Amazon.Lambda.Tools
export PATH="$PATH:/root/.dotnet/tools"
```
We can then use the `Amazon.Lambda.Tools` to package up the Lambda into a .zip ready to upload:
```
cd src/lambda21test
dotnet lambda package --msbuild-parameters "/p:PublishReadyToRun=true --self-contained false"
```

Lastly, we need to copy the .zip out of the Docker container onto our machines so we can upload it to Lambda via the AWS Console:
```
exit
docker start <containerId>
docker cp <containerId>:/app/src/lambda21test/bin/Release/netcoreapp3.1/lambda21test.zip lambda21test.zip
```

Apologies if any of that is clunky, I am not a Docker expert by any means.
