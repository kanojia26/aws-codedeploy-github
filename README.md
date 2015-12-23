# aws-codedeploy-github
Simple walk through for deploying a github-hosted project on EC2 Ubuntu instances using AWS CodeDeploy

#### Create 2 Roles:

1. For EC2 instances (CodeDeployInstanceRole):
    1. Select `Amazon EC2` and click on `Next Step`, then create the role.
    2. Add an Inline Policy: `{ "Version": "2012-10-17", "Statement": [ { "Action": [ "s3:Get*", "s3:List*" ], "Effect": "Allow", "Resource": "*" } ] }`
    3. Edit Trust Relationships: `{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" } ] }`
2. For CodeDeploy Application (CodeDeployTrustRole):
    1. Select `AWS CodeDeploy` and click on `Next Step`, then create the role.
    2. Add an Inline Policy: `{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": [ "autoscaling:CompleteLifecycleAction", "autoscaling:DeleteLifecycleHook", "autoscaling:DescribeAutoScalingGroups", "autoscaling:DescribeLifecycleHooks", "autoscaling:PutLifecycleHook", "autoscaling:RecordLifecycleActionHeartbeat", "ec2:DescribeInstances", "ec2:DescribeInstanceStatus", "tag:GetTags", "tag:GetResources" ], "Resource": "*" } ] }`
    3. Edit Trust Relationships: `{ "Version": "2012-10-17", "Statement": [ { "Action": "sts:AssumeRole", "Effect": "Allow", "Principal": { "Service": "codedeploy.amazonaws.com" } } ] }`

#### Launch EC2 Instances:
1. [Choose AMI tab]: Pick `Ubuntu Server`
2. [Configure Instance tab]: Pick the `CodeDeployInstanceRole` for IAM Role
3. [Configure Instance tab]: Click on `Advanced Details` and paste the following script into the box:
    ```
    #!/bin/bash

    apt-get -y update
    apt-get -y install awscli
    apt-get -y install ruby2.0
    cd /home/ubuntu
    aws s3 cp s3://aws-codedeploy-us-west-2/latest/install . --region us-west-2
    chmod +x ./install
    ./install auto
    ```
4. [Tag Instance tab]: Pick a tag (CodeDeployEC2Instance)
5. [Configure Security Group tab]: Add `HTTP` Rule
6. Create your instances

#### Grab a cup of tea
Wait till your EC2 instances are in `2/2 checks passed` state.

#### Create a CodeDeploy Application:
1. Pick `Custom Deployment`
2. Pick a name (MyApplication)
3. Pick a deployment group name (Production)
4. Pick `Name` as Key and `CodeDeployEC2Instance` as Value
5. Pick your desired configuration
6. Pick the Role ARN (ARN of CodeDeployTrustRole)
7. Create your application

#### Deploy a New Revision:
1. Click on `Deploy New Revision`
2. Pick an application (MyApplication)
3. Pick a deployment group (Production)
4. Pick `My application is stored in GitHub` for revision type
5. Connect to your github account
6. Put in your repository name (alibozorgkhan/aws-codedeploy-github)
7. Put in a commit version
8. Pick your desired configuration
9. Deploy

#### Check what you accomplished:
1. Check your EC2 instances' ip address. They all should display Apache2's default page.

#### Deploy from terminal
If you want to deploy using a terminal, you need to have a user that has CodeDeploy permissions. In order to do that, follow these steps:

1. On your AWS dashboard, go to `IAM`.
2. Create a new user called `Deployer` and store the credentials.
3. Go back to IAM dashboard and click on `Users`, then click on `Deployer`.
4. Under `Permissions` tab, click on `Attach Policy`.
5. Search for `AWSCodeDeployDeployerAccess` and attach the policy.

Now you can deploy using the created user. e.g. deploying using boto3 in python would be something like this:

```
import boto3
import os
import sys
from botocore.exceptions import ClientError

#   specify region
region = "us-west-2"

#   boto client
client = boto3.client('codedeploy', region_name=region)

#   application: development, stating, or production
application = sys.argv[1]

#   deployment group name: web server, worker, ...
deployment_group_name = sys.argv[2]

#   deploy!
try:
    response = client.create_deployment(
        applicationName=application,
        deploymentGroupName=deployment_group_name,
        revision={
            'revisionType': 'GitHub',
            'gitHubLocation': {
                'repository': 'YOUR_GITHUB_ORGANIZATION/YOUR_REPO_NAME',
                'commitId': 'COMMIT_ID'
            }
        },
        ignoreApplicationStopFailures=False
    )

    status = response.get('ResponseMetadata', {}).get('HTTPStatusCode')
    if status == 200:
        deployment_id = response.get('deploymentId')
        print "Deployment was successful: https://{0}.console.aws.amazon.com/codedeploy/home?region={0}#/deployments/{1}".format(
            region,
            deployment_id
        )
    else:
        raise Exception('Unknown error - {}\'s {}'.format(
            application, deployment_group_name
        ))
except ClientError, e:
    raise Exception('{} - {}\'s {}'.format(
        unicode(e), application, deployment_group_name
    ))
```