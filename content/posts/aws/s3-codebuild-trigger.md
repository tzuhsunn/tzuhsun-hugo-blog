---
title: "S3 Codebuild Trigger"
date: 2025-01-07T16:57:54+08:00 
lastmod: 2025-01-07T16:57:54+08:00 
categories:
- AWS
tags:
- AWS
- CI/CD
- AWS CDK
summary: "AWS S3 triggers CodeBuild"
description: "" 
weight: 
slug: ""
draft: false 
hidemeta: false
cover:
    image: "" 
    caption: ""
    alt: ""
    relative: false
---

# Overview
Code source is S3 file. Automatically trigger codebuild when S3 file is updated.
{{< figure  src="solution.png" title="sol" width="70%" align="center">}}
## Lambda
When the images in CN S3 bucket are updated, the lambda will wait until the file upload completely, and then trigger corresponding codebuild project.
## Codebuild, ECR, S3

# Content
## Use eventbridge
1. event rule
```json
{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["s3.amazonaws.com"],
    "eventName": ["PutObject", "CreateMultipartUpload", "CopyObject"],
    "requestParameters": {
      "bucketName": ["tomofun-images"]
    }
  }
}
```
Add codebuild as target of event rule (CDK)
```python
event_rule = aws_events.Rule(
    self,
    f"{service}-rule",
    rule_name=f"{service}-s3-trigger-codebuild",
    description=f"Trigger CodeBuild for {service} on S3 object update",
    event_pattern={
        "source": ["aws.s3"],
        "detail_type": ["AWS API Call via CloudTrail"],
        "detail": {
            "eventSource": ["s3.amazonaws.com"],
            "eventName": [
                "PutObject",
                "CreateMultipartUpload",
                "CopyObject",
            ],
            "requestParameters": {
                "bucketName": [f"{IMAGE_BUCKET}"],
                "key": [f"{file_name}"],
            },
        },
    },
)
event_rule.add_target(aws_events_targets.CodeBuildProject(project))
```

For codebuild buildspec, only can put json from AWS CDK.  
Originally in console, we directly add buildspec in yaml:
```yaml
version: 0.2

env:
  variables:
     SERVICE_NAME: test-jessie
phases:
  pre_build:
    commands:
      - IMAGE_REPO_NAME=${SERVICE_NAME}
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - docker load -i image/$SERVICE_NAME
      - docker images
        
      - echo "Selecting specific image"
      - DOCKER_IMAGE=$(docker images --format "{{.Repository}}:{{.Tag}}" | grep "^${SOURCE_ACCOUNT}" | grep "$IMAGE_REPO_NAME" | head -n 1)
      - echo "Selected DOCKER_IMAGE=$DOCKER_IMAGE"
      
      - GIT_HASH=$(echo $DOCKER_IMAGE | cut -d':' -f2)
      - echo "GIT_HASH=$GIT_HASH"
      - docker tag $DOCKER_IMAGE $ECS_SOURCE/$IMAGE_REPO_NAME:$GIT_HASH

  post_build:
    commands:
      - echo Push image...
      - docker push $ECS_SOURCE/$IMAGE_REPO_NAME:$GIT_HASH
```

Notice that if you want multiple lines command, use `|`.
```yaml
 - |
    while true; do
        echo "Checking file upload status...";
        FILE_LAST_MODIFIED=$(aws s3api head-object --bucket tomofun-images --key ${SERVICE_NAME}.zip --query 'LastModified' --output text --region $AWS_DEFAULT_REGION);
```

Be careful of column(`:`) usage in yaml:
https://stackoverflow.com/questions/61887429/yaml-file-error-message-expected-commands0-to-be-of-string-type
```yaml
  post_build:
    commands:
      - echo $buildStatus
      - buildComment=$(echo "Status of project build phase : $buildStatus")
```
`phase : $buildStatus"` part will lead to error: YAML_FILE_ERROR Message: Expected Commands[0] to be of string type:
  

In CDK, use online json-yaml converter. For example: https://onlineyamltools.com/convert-yaml-to-json. Note that mutiple lines command needs to be in one string.
```json
{
  "version": 0.2,
  "env": {
    "variables": {
      "SERVICE_NAME": "test-jessie",
    }
  },
  "phases": {
    "pre_build": {
      "commands": [
        "IMAGE_REPO_NAME=tomofun/${SERVICE_NAME}",
        "echo Logging in to Amazon ECR...",
        "$(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)",
        "while true; do\n  echo \"Checking file upload status...\";\n  FILE_LAST_MODIFIED=$(aws s3api head-object --bucket tomofun-images --key ${SERVICE_NAME}.zip --query 'LastModified' --output text --region $AWS_DEFAULT_REGION);\n  echo \"File Last Modified - $FILE_LAST_MODIFIED\";\n  if [[ \"$FILE_LAST_MODIFIED\" > \"$BUILD_START_TIME\" ]]; then\n      echo \"File is fully uploaded. Proceeding with the build.\";\n      break;\n  fi\n  if [[ \"$ELAPSED_TIME\" -ge \"$MAX_WAIT_TIME\" ]]; then\n      echo \"File upload not completed within the maximum wait time ($MAX_WAIT_TIME seconds). Exiting.\";\n      exit 1;\n  fi\n  echo \"File not fully uploaded yet. Waiting for $CHECK_INTERVAL seconds...\";\n  sleep $CHECK_INTERVAL;\n  ELAPSED_TIME=$((ELAPSED_TIME + CHECK_INTERVAL));\ndone\n"
      ]
    },
    "build": {
      "commands": [
        "docker load -i image/$SERVICE_NAME",
        "docker images",
      ]
    },
    "post_build": {
      "commands": [
        "echo Push image...",
        "docker push $ECS_SOURCE/$IMAGE_REPO_NAME:$GIT_HASH",
        "docker push $ECS_SOURCE/$IMAGE_REPO_NAME:staging"
      ]
    }
  }
}
```
Parse json to `buildspec` in CDK (codebuild.Project(buildspec=???))

## Use lambda
### Parse list to string as ENV
Lambda can only get string as env in CDK.
https://stackoverflow.com/questions/31352317/how-to-pass-a-list-as-an-environment-variable

###

## Determination
### [If I stream a file to s3, will the event trigger once the file is complete?](https://stackoverflow.com/questions/31297549/if-i-stream-a-file-to-s3-will-the-event-trigger-once-the-file-is-complete)
==> The answer is uncertain. This base on what event target/bridge/rule you use.
- For general cloudwatch event rule source, Amazon S3 provides read-after-write consistency for PUTS of new objects in your S3 bucket in all regions with one caveat. The caveat is that if you make a HEAD or GET request to the key name (to find if the object exists) before creating the object, Amazon S3 provides eventual consistency for read-after-write.
- That's why I give up using event rule, because I cannot wait for file being uploaded! And I cannot change the bodebuild stage sequence, which is, codebuild will directly download the source(s3) file once the event trigger! This will lead codebuild get old or no image, which is not preferrable :(
- However, when I try to use lambda and add the S3 bucket as its trigger, the function is only triggered until the file upload completely (WHO KNOWSSSSS)
- Other than lambda, SQS/SNS can also get `file completed` notification of S3. (https://stackoverflow.com/questions/32390263/get-notified-when-upload-is-completed-in-amazon-s3-bucket)
- How I guess the above: https://medium.com/@wei00925/%E5%9C%A8aws-lambda%E4%B8%AD%E4%BD%BF%E7%94%A8python%E8%87%AA%E5%8B%95%E8%99%95%E7%90%86s3%E4%B8%8A%E5%82%B3%E4%BA%8B%E4%BB%B6-dff68e9af093
- The `LastModified` attribute of s3 objects [head_object boto3 doc](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3/client/head_object.html)is the time you click upload, not the finishing time! Also, `LastModified` only changes once the file upates completely. E.g., The `LastModified` time is originally 2:30, and you click upload at 3:00. Before upload process completes, the query result of `LastModified` will be 2:30. Suppose the file completes at 3:05, the `LastModified` time will change to 3:00. (Notice: `LastModified` will be the start time rather than the end time 3:05.) 
