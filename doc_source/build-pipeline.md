# Building a continuous delivery pipeline for a Lambda application with AWS CodePipeline<a name="build-pipeline"></a>

You can use AWS CodePipeline to create a continuous delivery pipeline for your Lambda application\. CodePipeline combines source control, build, and deployment resources to create a pipeline that runs whenever you make a change to your application's source code\.

In this tutorial, you create the following resources\.
+ **Repository** – A Git repository in AWS CodeCommit\. When you push a change, the pipeline copies the source code into an Amazon S3 bucket and passes it to the build project\.
+ **Build project** – An AWS CodeBuild build that gets the source code from the pipeline and packages the application\. The source includes a build specification with commands that install dependencies and prepare an AWS Serverless Application Model \(AWS SAM\) template for deployment\.
+ **Deployment configuration** – The pipeline's deployment stage defines a set of actions that take the AWS SAM template from the build output, create a change set in AWS CloudFormation, and execute the change set to update the application's AWS CloudFormation stack\.
+ **AWS CloudFormation stack** – The deployment stage uses a template to create a stack in AWS CloudFormation\. The template is a YAML\-formatted document that defines the resources of the Lambda application\. The application includes a Lambda function and an Amazon API Gateway API that invokes it\.
+ **Roles** – The pipeline, build, and deployment each have a service role that allows them to manage AWS resources\. The console creates the pipeline and build roles when you create those resources\. You create the role that allows AWS CloudFormation to manage the application stack\.

The pipeline maps a single branch in a repository to a single AWS CloudFormation stack\. You can create additional pipelines to add environments for other branches in the same repository\. You can also add stages to your pipeline for testing, staging, and manual approvals\. For more information about AWS CodePipeline, see [What is AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html)\.

For an alternate method of creating a pipeline with AWS Serverless Application Model and AWS CloudFormation, watch [Automate your serverless application deployments](https://www.youtube.com/watch?v=0o3urdBeoII) on the Amazon Web Services YouTube channel\.

**Topics**
+ [Prerequisites](#with-pipeline-prepare)
+ [Create an AWS CloudFormation role](#with-pipeline-create-cfn-role)
+ [Set up a repository](#setup-repository)
+ [Create a pipeline](#create-pipeline1)
+ [Update the build stage role](#update-policy)
+ [Complete the deployment stage](#create-pipeline2)
+ [Test the application](#applications-pipeline-test)

## Prerequisites<a name="with-pipeline-prepare"></a>

This tutorial assumes that you have some knowledge of basic Lambda operations and the Lambda console\. If you haven't already, follow the instructions in [Getting started with AWS Lambda](getting-started.md) to create your first Lambda function\.

To follow the procedures in this guide, you will need a command line terminal or shell to run commands\. Commands are shown in listings preceded by a prompt symbol \($\) and the name of the current directory, when appropriate:

```
~/lambda-project$ this is a command
this is output
```

For long commands, an escape character \(`\`\) is used to split a command over multiple lines\.

On Linux and macOS, use your preferred shell and package manager\. On Windows 10, you can [install the Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10) to get a Windows\-integrated version of Ubuntu and Bash\.

During the build phase, the build script uploads artifacts to Amazon Simple Storage Service \(Amazon S3\)\. You can use an existing bucket, or create a new bucket for the pipeline\. Use the AWS CLI to create a bucket\.

```
$ aws s3 mb s3://lambda-deployment-artifacts-123456789012
```

## Create an AWS CloudFormation role<a name="with-pipeline-create-cfn-role"></a>

Create a role that gives AWS CloudFormation permission to access AWS resources\.

**To create an AWS CloudFormation role**

1. Open the [roles page](https://console.aws.amazon.com/iam/home#/roles) in the IAM console\.

1. Choose **Create role**\.

1. Create a role with the following properties\.
   + **Trusted entity** – **AWS CloudFormation**
   + **Permissions** – **AWSLambdaExecute**
   + **Role name** – **cfn\-lambda\-pipeline**

1. Open the role\. Under the **Permissions** tab, choose **Add inline policy**\. 

1. In **Create Policy**, choose the **JSON** tab and add the following policy\.

   ```
   {
       "Statement": [
           {
               "Action": [
                   "apigateway:*",
                   "codedeploy:*",
                   "lambda:*",
                   "cloudformation:CreateChangeSet",
                   "iam:GetRole",
                   "iam:CreateRole",
                   "iam:DeleteRole",
                   "iam:PutRolePolicy",
                   "iam:AttachRolePolicy",
                   "iam:DeleteRolePolicy",
                   "iam:DetachRolePolicy",
                   "iam:PassRole",
                   "s3:GetObject",
                   "s3:GetObjectVersion",
                   "s3:GetBucketVersioning"
               ],
               "Resource": "*",
               "Effect": "Allow"
           }
       ],
       "Version": "2012-10-17"
   }
   ```

## Set up a repository<a name="setup-repository"></a>

Create an AWS CodeCommit repository to store your project files\. For more information, see [Setting up](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up.html) in the CodeCommit User Guide\.

**To create a repository**

1. Open the [Developer tools console](https://console.aws.amazon.com/codesuite/home)\.

1. Under **Source**, choose **Repositories**\.

1. Choose **Create repository**\.

1. Follow the instructions to create and clone a repository named **lambda\-pipeline\-repo**\.

Create the following files in the repository folder\.

**Example index\.js**  
A Lambda function that returns the current time\.  

```
var time = require('time');
exports.handler = (event, context, callback) => {
    var currentTime = new time.Date();
    currentTime.setTimezone("America/Los_Angeles");
    callback(null, {
        statusCode: '200',
        body: 'The time in Los Angeles is: ' + currentTime.toString(),
    });
};
```

**Example template\.yml**  
The [AWS SAM template](gettingstarted-tools.md) that defines the application\.  

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Outputs the time
Resources:
  TimeFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: nodejs10.x
      CodeUri: ./
      Events:
        MyTimeApi:
          Type: Api
          Properties:
            Path: /TimeResource
            Method: GET
```

**Example buildspec\.yml**  
An [AWS CodeBuild build specification](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html) that installs required packages and uploads the deployment package to Amazon S3\. Replace the highlighted text with the name of your bucket\.  

```
version: 0.2
phases:
  install:
    runtime-versions:
        nodejs: 10
  build:
    commands:
      - npm install time
      - export BUCKET=lambda-deployment-artifacts-123456789012
      - aws cloudformation package --template-file template.yml --s3-bucket $BUCKET --output-template-file outputtemplate.yml
artifacts:
  type: zip
  files:
    - template.yml
    - outputtemplate.yml
```

Commit and push the files to CodeCommit\.

```
~/lambda-pipeline-repo$ git add .
~/lambda-pipeline-repo$ git commit -m "project files"
~/lambda-pipeline-repo$ git push
```

## Create a pipeline<a name="create-pipeline1"></a>

Create a pipeline that deploys your application\. The pipeline monitors your repository for changes, runs an AWS CodeBuild build to create a deployment package, and deploys the application with AWS CloudFormation\. During the pipeline creation process, you also create the AWS CodeBuild build project\.

**To create a pipeline**

1. Open the [Developer tools console](https://console.aws.amazon.com/codesuite/home)\.

1. Under **Pipeline**, choose **Pipelines**\.

1. Choose **Create pipeline**\.

1. Configure the pipeline settings and choose **Next**\.
   + **Pipeline name** – **lambda\-pipeline**
   + **Service role** – **New service role**
   + **Artifact store** – **Default location**

1. Configure source stage settings and choose **Next**\.
   + **Source provider** – **AWS CodeCommit**
   + **Repository name** – **lambda\-pipeline\-repo**
   + **Branch name** – **master**
   + **Change detection options** – **Amazon CloudWatch Events**

1. For **Build provider**, choose **AWS CodeBuild**, and then choose **Create project**\.

1. Configure build project settings and choose **Continue to CodePipeline**\.
   + **Project name** – **lambda\-pipeline\-build**
   + **Operating system** – **Ubuntu**
   + **Runtime** – **Standard**
   + **Runtime version** – **aws/codebuild/standard:2\.0**
   + **Image version** – **Latest**
   + **Buildspec name** – **buildspec\.yml**

1. Choose **Next**\.

1. Configure deploy stage settings and choose **Next**\.
   + **Deploy provider** – **AWS CloudFormation**
   + **Action mode** – **Create or replace a change set**
   + **Stack name** – **lambda\-pipeline\-stack**
   + **Change set name** – **lambda\-pipeline\-changeset**
   + **Template** – **BuildArtifact::outputtemplate\.yml**
   + **Capabilities** – **CAPABILITY\_IAM**, **CAPABILITY\_AUTO\_EXPAND**
   + **Role name** – **cfn\-lambda\-pipeline**

1. Choose **Create pipeline**\.

The pipeline fails the first time it runs because it needs additional permissions\. In the next section, you add permissions to the role that's generated for your build stage\.

## Update the build stage role<a name="update-policy"></a>

During the build stage, AWS CodeBuild needs permission to upload the build output to your Amazon S3 bucket\.

**To update the role**

1. Open the [roles page](https://console.aws.amazon.com/iam/home#/roles) in the IAM console\.

1. Choose **code\-build\-lamba\-pipeline\-service\-role**\.

1. Choose **Attach policies**\.

1. Attach **AmazonS3FullAccess**\.

## Complete the deployment stage<a name="create-pipeline2"></a>

The deployment stage has an action that creates a change set for the AWS CloudFormation stack that manages your Lambda application\. A change set specifies the changes that are made to the stack, such as adding new resources and updating existing resources\. Change sets let you preview the changes that are made before making them, and add approval stages\. Add a second action that executes the change set to complete the deployment\.

**To update the deployment stage**

1. Open your pipeline in the [Developer tools console](https://console.aws.amazon.com/codesuite/home)\.

1. Choose **Edit**\.

1. Next to **Deploy**, choose **Edit stage**\.

1. Choose **Add action group**\.

1. Configure deploy stage settings and choose **Next**\.
   + **Action name** – **execute\-changeset**
   + **Action provider** – **AWS CloudFormation**
   + **Input artifacts** – **BuildArtifact**
   + **Action mode** – **Execute a change set**
   + **Stack name** – **lambda\-pipeline\-stack**
   + **Change set name** – **lambda\-pipeline\-changeset**

1. Choose **Done**\.

1. Choose **Save**\.

1. Choose **Release change** to run the pipeline\.

Your pipeline is ready\. Push changes to the master branch to trigger a deployment\.

## Test the application<a name="applications-pipeline-test"></a>

The application includes an API Gateway API with a public endpoint that returns the current time\. Use the Lambda console to view the application and access the API\.

**To test the application**

1. Open the Lambda console [Applications page](https://console.aws.amazon.com/lambda/home#/applications)\.

1. Choose **lambda\-pipeline\-stack**\.

1. Under **Resources**, expand **ServerlessRestApi**\.

1. Choose **Prod API endpoint**\.

1. Add **/TimeResource** to the end of the URL\. For example, `https://l193nqxdjj.execute-api.us-east-2.amazonaws.com/Prod/TimeResource`\.

1. Open the URL\.

The API returns the current time in the following format\.

```
The time in Los Angeles is: Thu Jun 27 2019 16:07:20 GMT-0700 (PDT)
```