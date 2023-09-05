---
layout: post
title: "CDK CI/CD: introduction"
categories: aws cdk cicd
date: 2019-08-19 18:00:00 +0200
tags:
    - aws
    - codepipeline
    - codebuild
    - cdk
    - CI/CD
---

# CI/CD in AWS with CDK: Introduction

> In late 2019, I started experimenting around with the CDK. The use of a programming language enables you to overcome various hurdles, as it grants you a set of control flow elements such as for-loops, if-else statements and more.

In July 2019, the AWS CDK (Cloud Development Kit) was released as a code-first approach to provisioning cloud application resources. The use of Typescript, Python, Java and a growing number of bindings for other languages presented a solution to the pile-up of verbose CloudFormation (CF) code. Especially in big projects, the amount of near-identical resources with only slightly different parameters can lead to maintenance hell.

That's not to say there weren't other options available. Some open-source projects like [Terraform](https://www.terraform.io/) provide control flow mechanisms. DIY coders could write a [transpiler](https://devopedia.org/transpiler), churning out CloudFormation code ready for deployment.

Another course of action would be to implement [custom CloudFormation macros](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-macros.html) for processing your CF template. The serverless application model (SAM) is a well known example. Furthermore, macros can be chained as transforms in your CF template. Each transform takes the previous CF template, processes it and hands the resulting template over to the next transform. After the last transform the CF template is submitted for deployment. An example for a macro can be found [here](https://github.com/avanderm/aws-cloudformation-macros).

One downside of macros however, depending on what logic is required, is the need to invent syntax that you will then use in your unprocessed CF template. Once requirements change or additional features are desired you will either have to be damn certain they are backwards compatible or you would have to version the macros. Once in use it can be difficult to maintain an overview of stacks reliant upon a macro.

In late 2019, I started experimenting around with the CDK. The use of a programming language enables you to overcome various hurdles, as it grants you a set of control flow elements such as for-loops, if-else statements and more. Situations that rise in more complex architecture become easier to tackle:

- Similar resources or sets thereof with different parameters. The parameter sets can be stored in a configuration file and combined with a base resource.
- CDK's notion of a construct allows for rapidly switching resources around from stacks to nested stacks or absorb it into already existing stacks. Constructs makes for reusable components, but with more benefits and options than nested stacks ever could.
- Aggregating resources/metrics can be done without the headache of having to make changes in several templates. For example, in our application we grouped a bunch of CloudWatch metrics, located in different stacks, into a central dashboard for visualisation.

In order to effectively use the CDK, it requires a change in mentality when it comes to stack design. With the CDK, most of the work is done during the synthesis of the CloudFormation code. This allows to catch (most) errors early on so you can iterate faster, and faulty deployments won't east up as much time. Compare it to compiled languages vs interpreted languages if you will. If more mistakes are detected at "synthesis" time, this will save you time deploying stacks through trial and error.

Adhering to the principles of DevOps, one question that has been bugging me is how to best fit in the CDK when designing CI/CD pipelines. In this blog post I hope to cover some options in a hands-on manner, with plenty of code to inspect. In addition, some interesting caveats are covered for those that wish to start using the CDK.

## Setup

The source code for the example presented in this post is split up into a [code repository](https://github.com/avanderm/aws-cicd-docker), containing the business logic of an application, and an [infrastructure repository](https://github.com/avanderm/aws-cicd-cdk), containing the CDK project. Deployment instructions are included on Github. As expected, most of the discussion will revolve around the latter repository.

The business logic will run in a ECS service, it comprises a simple process listening to a SQS queue and validating individual messages. Multiple such queue/service combinations can be spun up using a configuration file any team member can modify to their liking:

{% highlight yaml %}
processor1:
  ageRestriction: 28
  version: latest

processor2:
  ageRestriction: 31
{% endhighlight %}
*SQS messages contain persons with an age field, if the age below the age restriction, it is discarded.*

A mix of engineer and non-engineer variables are made available, and can be expanded upon. Each processor will have its own queue and ECS service. Messages in the queue correspond to people with an age field. Is their age above the restriction, they pass, otherwise they do not. Simple.

The second (optional) variable specifies the version (tag) of the image in ECR you wish to you. Perhaps big changes to the business logic are being made and for some processors you wish to freeze the image version. The main point here is: some variables may have an effect on business logic, while others may have an effect on architecture (e.g. how much CPU, memory, ...).

We will look into three different CDK implementations. They do have a degree of overlap. For example, all three implementations make use of three CI/CD pipelines. Two of those are common in every implementation:

1. A CI/CD pipeline listening to the business logic repository will build the image for the ECR repository on every update to master and tag it as latest.
2. A CI/CD pipeline listening to changes to the image in ECR tagged latest will update all ECS services that make use of the latest image. We will call this one the ECS redeployment stack.

Lastly, we include a stack with a CloudWatch dashboard, to group together SQS queue metrics for the purpose of monitoring. With that said, we cover the first implementation in the [next installment](/aws/cdk/cicd/2019-08-19-nested-stacks).
