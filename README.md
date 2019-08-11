# securityhub-remediations
Workshop for implementing rmediations using AWS Security Hub and Cloud Custodian

# Overview
In this workshop you will learn how to implement automated remediations of findings submitted to Security Hub.


* Level: Intermediate
* Duration: 1:30 - 2:00 hours
* CSF Functions: Detect, Respond
* CAF Components: Detective, Responsive

# Prerequisites

1. You will need an AWS account for this workshop and administrative credentials, with console and aws cli access. 
2. We strongly suggest you use a new/clean account, or at least one in which can tolerate the terminatation, stopping, and/or deleting of resources.
3. You will incur charges for the AWS resources used in this workshop. The charges for some of the resources may be covered through the AWS Free Tier. The demo uses free tier choices wherever possible.
4. You must run this workshop in a region supported by AWS Security Hub (https://docs.aws.amazon.com/general/latest/gr/rande.html#sechub_region).  We recommended using the us-east-1 region.
5. You must run this workshop in a region support by AWS Cloud9 (https://docs.aws.amazon.com/general/latest/gr/rande.html#cloud9_region), 
or be comfortable setting up a python3 environment with pip3, ssh, and any text editor.
6. You should already have GuardDuty enabled on the account, if not follow https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_settingup.html#guardduty_enable-gd 

# Modules

1. {replace with final model names}


## Module 1 - Environment Build and Configuration
1. Enable Security Hub (if not already enabled - https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-settingup.html#securityhub-enable
2. Create a Cloud9 Environment for this Workshop 
2.1. Open https://us-east-1.console.aws.amazon.com/cloud9/home?region=us-east-1
2.2. Click Create environment
2.3. In the name field, type "SecHubWorkshop" then click "Next step"
2.4. On the "Configure settings" page, click "Next step"
2.5. On the "Review" page, click "Create environment"
3. Create an IAM Policies and Roles
3.1. Creating a policy for the Cloud9 EC2 Instance
3.1.1. Open https://console.aws.amazon.com/iam/home?region=us-east-1#/policies
3.1.2. Click "Create policy"
3.1.3. Click the "JSON" tab
3.1.4. Replace the prepopulated text with the following:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "guardduty:CreateSampleFindings",
            "Resource": "arn:aws:guardduty:*:*:detector/*"
        }
    ]
}
```
3.1.5. Click "Review Policy"
3.1.6. In the Name field, enter "Cloud9RemediationTesting"
3.1.7. Click "Create Policy"
3.2. Creating a policy for CloudCustodian 
3.2.1. Click "Create policy"
3.2.2. Click the "JSON" tab
3.2.3. Replace the prepopulated text with the following:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CCWrite",
            "Effect": "Allow",
            "Action": [
                "ssm:SendCommand",
                "ssm:CreateOpsItem",
                "iam:DeleteAccessKey",
                "iam:UpdateAccessKey",
                "iam:Tag*",
                "iam:UnTag*",
                "kms:UntagResource",
                "kms:TagResource",
                "events:PutRule",
                "events:PutTargets",
                "ec2:CreateImage",
                "ec2:CreateSnapshot*",
                "ec2:CreateTags",
                "ec2:DeleteTags",
                "ec2:ModifyVpcAttribute",
                "ec2:ModifyInstanceAttribute",
                "ec2:AssociateIamInstanceProfile",
                "ec2:DisassociateIamInstanceProfile",
                "ec2:StopInstances",
                "ec2:TerminateInstances",
                "config:PutEvaluations",
                "s3:PutBucketTagging",
                "tag:TagResources",
                "tag:UntagResources"
            ],
            "Resource": "*"
        },
        {
            "Sid": "CCDeployLambdas",
            "Effect": "Allow",
            "Action": [
                "lambda:CreateFunction",
                "lambda:TagResource",
                "lambda:InvokeFunction",
                "lambda:UpdateFunctionConfiguration",
                "lambda:UntagResource",
                "lambda:UpdateAlias",
                "lambda:UpdateFunctionCode",
                "lambda:AddPermission",
                "lambda:DeleteFunction",
                "lambda:RemovePermission",
                "lambda:CreateAlias",
                "lambda:GetFunction"
            ],
            "Resource": [
                "arn:aws:lambda:*:{AWS_ACCOUNT_NUMBER}:function:custodian-*"
            ]
        },
        {
            "Sid": "CCPassRole",
            "Effect": "Allow",
            "Action": [
                "iam:PassRole"
            ],
            "Resource": [
                "arn:aws:iam::{AWS_ACCOUNT_NUMBER}:role/CloudCustodian"
            ]
        },
        {
            "Sid": "CCSecurityHub",
            "Effect": "Allow",
            "Action": [
                "securityhub:UpdateFindings",
                "securityhub:BatchImportFindings",
                "securityhub:CreateActionTarget",
                "securityhub:UpdateActionTarget",
                "securityhub:Describe*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "CCLogs",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:CreateLogGroup"
            ],
            "Resource": "arn:aws:logs:*:{AWS_ACCOUNT_NUMBER}:log-group:/aws/lambda/custodian-*"
        }
    ]
}
```
3.2.4. Replace "{AWS_ACCOUNT_NUMBER}" with your AWS Account number, otherwise you will get validation errors on the next step.
3.2.5. Click "Review Policy"
3.2.6. In the Name field, enter "CloudCustodian"
3.2.7. Click "Create Policy"
3.3. Creating a Role for the Cloud9 EC2 Instance
3.3.1. Click "Create Role"
3.3.2. Under "Choose the service that will use this role, click "EC2" then "Next: Permissions"
3.3.3. In the "Filter Policies" searchbox, Enter "Cloud9RemediationTesting" then hit return.
3.3.4. Click the checkbox for the "Cloud9RemediationTesting" policy
3.3.5. Click "Next: Tags"
3.3.6. Click "Next: Review"
3.3.7. In "Role name", enter "Cloud9Instance"
3.3.8. Click "Create role"
3.4. Creating a Role for the Cloud Custodian
3.4.1. Click "Create Role"
3.4.2. Under "Choose the service that will use this role, click "Lambda" then "Next: Permissions"
3.4.3. In the "Filter Policies" searchbox, Enter "CloudCustodian" then hit return.
3.4.4. Click the checkbox for the "CloudCustodian" policy
3.4.5. In the "Filter Policies" searchbox, Replace the text in the "Filter Policies" searchbox with "SecurityAudit".  
3.4.6. Click the checkbox next to "SecurityAudit", and if more than one appears, select the one with the AWS Managed Policies icon (an orange cube)
3.4.7. Click "Next: Tags"
3.4.5. Click "Next: Review"
3.4.6. In "Role name", enter "CloudCustodian"
3.4.7. Click "Create role" 
3.4.8. Click on the "CloudCustodian" role that was just created.
3.4.9. Click on the "Trust relationships" tab.
3.4.10. Click on "Edit trust relationship"
3.4.11. Replace the prepopulated text with the following, and replace the "{AWS_ACCOUNT_NUMBER}" with your AWS account id.  If 
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com",
        "AWS": "arn:aws:iam::{AWS_ACCOUNT_NUMBER}:role/Cloud9RemediationTesting"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
5. Setup AWS cli on Cloud9 environment
5.1. metadata file
6. Install Cloud Custodian
7. Test Cloud Custodian

## Module 2 - Security Hub Custom Actions - Human initiated automation

## Module 3 - Automated Remediations - GuardDuty Event on EC2 Instance

## Module 4 - Automated Remediations - GuardDuty Event on IAMUser

## Module 5 - Automated Remediations - GuardDuty Event on EC2 Instance - Isolate rather than Stop



