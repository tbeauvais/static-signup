# AWS Static Signup Form Project

Prerequisites:
  AWS Free account
  GitHub public account

Create git project (my-signup-form)

1) create index.html that includes sign-up form
2) manually create an s3 bucket
      make static website
3) copy index.html to s3 bucket
4) find URL and launch in Browser
    - is there a problem?

1) Add pipeline.yml for for creating S3 Bucket

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

2) Run Create Stack in CloudFormation using your current login.

3) Delete Stack

4) Create service role used for running CloudFormation templates.
    Use Console to create service role
    Give access to S3 (create, delete)
    Review Edit Trust Relationship
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





