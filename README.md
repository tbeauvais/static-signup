# AWS Static Signup Form Project

## Prerequisites
* Signup for AWS Free account
* GitHub public account
* Access to Signup Forms

## Step 1
* Create git project (my-signup-form)
* Add index.html that includes a sign-up form

## Step 2
* Use AWS Console to create a new S3 bucket 
  * Make S3 bucket a static website (under bucket properties)
* Upload index.html to S3 bucket
   * make index.html file public read
* Find URL and launch in Browser (under bucket properties/Static website hosting)

## Step 3
* Delete the S3 bucket you created in step 2

## Step 4
We will now stat to automate the creation of infrastructure 

* Add a file named pipeline.yml to your github project with the following content

http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-s3-bucket.html
```
AWSTemplateFormatVersion: '2010-09-09'
Description: Pipeline for building pipeline for signup form S3 static website
Parameters:
  SiteBucketName:
    Type: String
    Description: Name of bucket to create to host the website
    Default: "tomb-pipeline-static-site"
Resources:
  SiteBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Delete
    Properties:
      AccessControl: PublicRead
      BucketName: !Ref SiteBucketName
      WebsiteConfiguration:
        IndexDocument: index.html
```


* Run Create Stack in CloudFormation using your current login. Use the pipeline.yml as the template.
  * Verify the S3 bucket was created

* Run Delete Stack in CloudFormation for the stack you just created

## Step 5
Here we will create a new IAM Role that we will use from now on when running cloudformation operations.

* Create service role used for running CloudFormation templates.
  * Use Console to create service role
  * Give access to S3 (create, delete). Attach AmazonS3FullAccess Policy
  * Review Edit Trust Relationship
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudformation.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
    Review Permissions
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "*"
        }
    ]
}
```

5) Run Create Stack in CloudFormation using new Role.


6) Add CodeBuildRole and CodePipelineRole to cloud formation pipeline.yml template
```
  CodeBuildRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - codebuild.amazonaws.com
          Action:
          - sts:AssumeRole
      Policies:
      - PolicyName: codebuild-service
        PolicyDocument:
          Statement:
          - Action:
            - logs:PutLogEvents
            - logs:CreateLogGroup
            - logs:CreateLogStream
            - s3:GetObject
            - s3:GetObjectVersion
            - s3:PutObject
            - s3:PutObjectAcl
            Resource: "*"
            Effect: Allow
          Version: '2012-10-17'
  CodePipelineRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - codepipeline.amazonaws.com
          Action:
          - sts:AssumeRole
      Policies:
      - PolicyName: codepipeline-service
        PolicyDocument:
          Statement:
          - Action:
            - codebuild:*
            Resource: "*"
            Effect: Allow
          - Action:
            - s3:GetObject
            - s3:GetObjectVersion
            - s3:GetBucketVersioning
            Resource: "*"
            Effect: Allow
          - Action:
            - s3:PutObject
            Resource:
            - arn:aws:s3:::codepipeline*
            Effect: Allow
          - Action:
            - s3:*
            - cloudformation:*
            - iam:PassRole
            Resource: "*"
            Effect: Allow
          Version: '2012-10-17'
```


6) Create roles for pipeline and build
http://docs.aws.amazon.com/codebuild/latest/userguide/how-to-create-pipeline.html
http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-iam-role.html

Create Policy for creating roles, and attach to role
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1506012004000",
            "Effect": "Allow",
            "Action": [
                "iam:AttachRolePolicy",
                "iam:CreatePolicy",
                "iam:CreateRole",
                "iam:DeletePolicy",
                "iam:DeleteRole",
                "iam:PutRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRole",
                "iam:PassRole"
            ],
            "Resource": [
                "arn:aws:iam::651438405399:role/*"
            ]
        },
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": [
                "codebuild:CreateProject",
                "codebuild:DeleteProject",
                "codebuild:UpdateProject",
                "codepipeline:CreatePipeline",
                "codepipeline:DeletePipeline",
                "codepipeline:GetPipeline",
                "codepipeline:GetPipelineState"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

8) Add CodeBuild resource and pipeline bucket

Add new parameters for github options

```
  GitHubUser:
    Type: String
    Description: GitHub User
    Default: "tbeauvais"
  GitHubRepo:
    Type: String
    Description: Name of GitHub repo
    Default: "static-signup"
  GitHubToken:
    NoEcho: true
    Type: String
    Description: OAuthToken access token for repo. See https://github.com/settings/tokens
```

Add pipeline bucket (name will be generated)
```
  PipelineBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Delete
```

Add
```
  CodeBuildDeploySite:
    Type: AWS::CodeBuild::Project
    DependsOn: CodeBuildRole
    Properties:
      Name: !Sub ${AWS::StackName}-DeploySite
      Description: Deploy signup form site to S3
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Artifacts:
        Type: CODEPIPELINE
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: "aws/codebuild/ubuntu-base:14.04"
      Source:
        Type: CODEPIPELINE
        BuildSpec: !Sub |
          version: 0.1
          phases:
            post_build:
              commands:
                - ls
                - aws s3 cp --acl public-read ./index.html s3://${SiteBucketName}/
          artifacts:
            type: zip
            files:
              - ./index.html
      TimeoutInMinutes: 10
```


```
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      RoleArn: !GetAtt CodePipelineRole.Arn
      Stages:
      - Name: Source
        Actions:
        - InputArtifacts: []
          Name: Source
          ActionTypeId:
            Category: Source
            Owner: ThirdParty
            Version: '1'
            Provider: GitHub
          OutputArtifacts:
          - Name: SourceOutput
          Configuration:
            Owner: !Ref GitHubUser
            Repo: !Ref GitHubRepo
            Branch: master
            OAuthToken: !Ref GitHubToken
          RunOrder: 1
      - Name: Deploy
        Actions:
        - Name: Artifact
          ActionTypeId:
            Category: Build
            Owner: AWS
            Version: '1'
            Provider: CodeBuild
          InputArtifacts:
          - Name: SourceOutput
          OutputArtifacts:
          - Name: DeployOutput
          Configuration:
            ProjectName: !Ref CodeBuildDeploySite
          RunOrder: 1
      ArtifactStore:
        Type: S3
        Location: !Ref PipelineBucket
```


# Next Steps
Move BuildSpec commands from inline to buildspec.yml file (http://docs.aws.amazon.com/codebuild/latest/userguide/getting-started.html#getting-started-create-build-spec)
Restructure project (move site/html under folder)
Add simple test
Add aprover stage
