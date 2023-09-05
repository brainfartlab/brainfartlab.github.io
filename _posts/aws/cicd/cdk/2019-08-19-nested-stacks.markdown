---
layout: post
title: "CDK CI/CD: nested stacks approach"
categories: aws cdk cicd
date: 2019-08-19 19:00:00 +0200
tags:
    - aws
    - codepipeline
    - codebuild
    - cdk
    - CI/CD
---

# CI/CD in AWS with CDK: Nested Stacks

> Switching from CloudFormation to the CDK takes some getting used to. Pre-CDK we would commonly employ Makefiles to deploy stacks. Since dealing with separate stacks can be annoying, especially when dependencies are involved, anchoring dependent stacks as nested stacks tends to be a common practice.

All code can be found in the Github repositories for [infrastructure](https://github.com/avanderm/aws-cicd-cdk) and [code](https://github.com/avanderm/aws-cicd-docker).

Switching from CloudFormation to the CDK takes some getting used to. Pre-CDK we would commonly employ Makefiles to deploy stacks. Since dealing with separate stacks can be annoying, especially when dependencies are involved, anchoring dependent stacks as nested stacks tends to be a common practice.

This implementation can be seen as a naive attempt when one is new to the CDK. Nevertheless, even in this implementation we will see some interesting tips. This is how the CI/CD pipeline looks like:

![Nested stack pipeline](/assets/placeholder.png){:class="img-responsive"}
*Nested stack approach: the ECS redeployment stack and the dashboard stack are nested stacks.*

In short, there are three stacks: (1) the stack for this pipeline itself, (2) DockerPipeline - the stack for building the ECR image and (3) the main stack, which contains the pipeline for deploying changes to the latest image in ECR to ECS and the dashboard stack as nested stacks.

After the CF templates have been synthesized, the pipeline updates itself before any others. This is to ensure that any changes to the flow of the pipeline are deployed first. If such a change occurs, the pipeline restart. The pipeline restart is really only needed when additional CodePipeline stages and actions are added taking position before the self-update, since they would otherwise be skipped.

Because self updating causes the CF stack for the pipeline to use its own IAM role, we include the necessary IAM permission inline as to prevent the pipeline from being unable to delete itself. If it is not set to inline, the CDK will create a policy. Upon deletion the IAM role would then lose the policy and that renders it unable to delete itself. We include it here:


{% highlight ts %}
const selfDeploymentRole = new iam.Role(this, 'TaskRole', {
  assumedBy: new iam.ServicePrincipal('cloudformation.amazonaws.com'),
  inlinePolicies: {
    'self-destruct': new iam.PolicyDocument({
      statements: [
        iamRolePermissions,
        iamPolicyPermissions
      ]
    })
  }
});
{% endhighlight %}

Making a working pipeline required more knowledge about the intricacies of CDK. The next sections will go in depth about some CDK behaviours that do not present themselves in an obvious way.

## CDK and external resources

Some external resources are shared among several stacks, such as the artifact bucket for the CI/CD pipelines, and the VPC network for the ECS services. Rather than repeating code, a dummy stack can be used to represent these resources. The object representations can then be used by others stacks.

It would be more straightforward to set them up directly in the application code file, but CDK requires that resources belong to a stack. You might find that with manual deployment using the CDK the dummy stack will show up in your AWS account, be it with no resources (only the `AWS::CDK::Metadata` resource).

## CDK creates stack parameters for nested stacks

The CodeBuild step has the duty to synthesize the CF templates, which are then used in the deployment steps. A quick glance to the [buildspec](https://github.com/avanderm/aws-cicd-cdk/blob/nested-stacks/buildspec.yml) file reveals some additional logic to the synthesis step:

{% highlight yaml %}
post_build:
    commands:
      - cat templateConfiguration.json | jq --arg env $ENVIRONMENT '.Tags.Environment = $env' | sponge templateConfiguration.json
      - jq -f scripts/extract_assets.jq --arg stack MainStack $BUILD_DIR/manifest.json > assets.json
      - python scripts/upload_assets.py
      - jq -s '.[0] * .[1]' basicConfiguration.json mainConfiguration.json | sponge mainConfiguration.json
{% endhighlight %}

These post synthesis actions will create a template configuration to be passed in the CF deployment steps in CodePipeline. [Template configurations](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/continuous-delivery-codepipeline-cfn-artifacts.html#w2ab1c13c17c15) allow you to pass stack tags and parameters to your stack deployments.

In our case we pass some tags and setting the environment tag during the build. But we also pass parameters to the main stack. Yet the CDK has a clear preference to avoid parameters; at synthesis time the template are fixed and passing parameters at deployment time are not needed. So what is going on?

The explanation lies in how the CDK works with nested stacks. For every nested stack, three stack parameters are introduced in the parent stack: a bucket, a version key and an artifact hash. The combination of the bucket and version key point to a location in S3 where the main stack expects to find the template for the nested stack. The artifact hash is there to verify MD5 integrity. After more inspection we learn the following:

- The way the CDK deals with nested stacks is to introduce the aforementioned three parameters into the parent stack. Deploying with the CDK will upload the templates of the nested stack and substitute in the values at deployment time. This puts a limit to the number of nested stacks to 20, since a CF template can have no more than 60 parameters at the time of writing.
- In the CDK build folder we find not only the generated CF templates, but also a file named `manifest.json`. This file keeps tracks of stacks tags, environments, stack traces and more. One of its duties is to list assets for stacks, in our case the nested stack templates. We will rely on this file to fill in the blanks when CodePipeline takes over from the CDK, since the parameter names for the parent stack are generated at synthesis time.

Here's an example of an asset in the manifest:

{% highlight json %}
{
  "artifacts": {
    "MainStack": {
      "metadata": {
        "/MainStack": [
          {
            "type": "aws:cdk:asset",
            "data": {
              "path": "MainStackDeployPipeline3909F365.nested.template.json",
              "id": "d9b6f9056801a4041ee81efe14d70fd893e8d4591a37f61aac0b04d56684d9f4",
              "packaging": "file",
              "sourceHash": "d9b6f9056801a4041ee81efe14d70fd893e8d4591a37f61aac0b04d56684d9f4",
              "s3BucketParameter": "AssetParametersd9b6f9056801a4041ee81efe14d70fd893e8d4591a37f61aac0b04d56684d9f4S3Bucket948AE1F2",
              "s3KeyParameter": "AssetParametersd9b6f9056801a4041ee81efe14d70fd893e8d4591a37f61aac0b04d56684d9f4S3VersionKeyC9BA9383",
              "artifactHashParameter": "AssetParametersd9b6f9056801a4041ee81efe14d70fd893e8d4591a37f61aac0b04d56684d9f4ArtifactHash5C2CFACC"
            }
          }
        ]
      }
    }
  }
}
{% endhighlight %}

Based off of this file we know what parameter names are assigned for the bucket, version key and artifact hash, and we can extract them (post build command 2) with the help of a tool such as jq.

Because the CF template for the nested stack also need to be in S3 for the parent stack to find, a Python script uploads it to S3 and formats the stack parameters (post build command 3) so that they can be merged with the desired stack tags (post build command 4).

## Declarative aspects of the CDK

People with a keen eye might have noticed something unnecessary at first glance in passing the ECR repository to the ECS deployment pipeline.


{% highlight ts %}
new EcsStack(this, 'DeployPipeline', {
  imageRepositoryName: props.repository.repositoryName,
  ecsServices: listeningServices,
  artifactBucket: props.artifactBucket
});
{% endhighlight %}

We pass not the ECR repository object itself but rather the name. Why? The CDK is smart enough to deal with objects. You can also see that this causes extra code in the ECS pipeline implementation:

{% highlight ts %}
new codepipeline_actions.EcrSourceAction({
  actionName: 'Image',
  output: imageOutput,
  repository: ecr.Repository.fromRepositoryName(this, 'Repository', props.imageRepositoryName),
  imageTag: props.tag ? props.tag : 'latest'
})
{% endhighlight %}

If we were to refactor this code and simply pass the repository object into the ECS pipeline implementation, the code would look as follows:

{% highlight ts %}
new codepipeline_actions.EcrSourceAction({
  actionName: 'Image',
  output: imageOutput,
  repository: repository,
  imageTag: props.tag ? props.tag : 'latest'
})
{% endhighlight %}

However, it would also not synthesize. Here's why: behind the scenes the CDK will generate the CF template, which includes an `AWS::Events::Rule` object. This rule will pick up changes to the latest image in ECR and propagate it towards its target, which is the pipeline in the ECS redeployment stack. If we choose the simpler code solution, this rule however will be generated in the DockerPipeline stack. This in turn creates a cyclical dependency. The DockerPipeline stack needs to be deployed before the pipeline of the ECS redeployment stack nested stack (and therefore the MainStack), since the latter requires a name export of the ECR repository. Yet the DockerPipeline stack demands an export of the ECS deployment pipeline, because it is the target of the rule. We are stuck.

But not if we pass the repository name; in that case the rule is also generated, but this time in the ECS redeployment stack, as it should. Something to keep in mind when passing objects or properties thereof and you run into cyclical dependency alerts.

In the [next installment](/aws/cdk/cicd/2019-08-19-separate-stacks), we will gravitate away from the naive approach and play more to the strengths of the CDK by applying divide-and-conquer to our stacks.
