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
    1. Open https://us-east-1.console.aws.amazon.com/cloud9/home?region=us-east-1
    2. Click Create environment
    3. In the name field, type "SecHubWorkshop" then click "Next step"
    4. On the "Configure settings" page, click "Next step"
    5. On the "Review" page, click "Create environment"
3. Creating an IAM Policy for the Cloud9 EC2 Instance
    1. Open https://console.aws.amazon.com/iam/home?region=us-east-1#/policies
    2. Click "Create policy"
    3. Click the "JSON" tab
    4. Replace the prepopulated text with the following:
```json
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
    5. Click "Review Policy"
    6. In the Name field, enter "Cloud9RemediationTesting"
    7. Click "Create Policy"
4. Creating an IAM Policy for CloudCustodian 
    1. Click "Create policy"
    2. Click the "JSON" tab
    3. Replace the prepopulated text with the following:
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
    4. Replace "{AWS_ACCOUNT_NUMBER}" with your AWS Account number, otherwise you will get validation errors on the next step.
    5. Click "Review Policy"
    6. In the Name field, enter "CloudCustodian"
    7. Click "Create Policy"
5. Creating a Role for the Cloud9 EC2 Instance
    1. Click "Create Role"
    2. Under "Choose the service that will use this role, click "EC2" then "Next: Permissions"
    3. In the "Filter Policies" searchbox, Enter "Cloud9RemediationTesting" then hit return.
    4. Click the checkbox for the "Cloud9RemediationTesting" policy
    5. Click "Next: Tags"
    6. Click "Next: Review"
    7. In "Role name", enter "Cloud9Instance"
    8. Click "Create role"
6. Creating a Role for the Cloud Custodian
    1. Click "Create Role"
    2. Under "Choose the service that will use this role, click "Lambda" then "Next: Permissions"
    3. In the "Filter Policies" searchbox, Enter "CloudCustodian" then hit return.
    4. Click the checkbox for the "CloudCustodian" policy
    5. In the "Filter Policies" searchbox, Replace the text in the "Filter Policies" searchbox with "SecurityAudit".  
    6. Click the checkbox next to "SecurityAudit", and if more than one appears, select the one with the AWS Managed Policies icon (an orange cube)
    7. Click "Next: Tags"
    5. Click "Next: Review"
    6. In "Role name", enter "CloudCustodian"
    7. Click "Create role" 
    8. Click on the "CloudCustodian" role that was just created.
    9. Click on the "Trust relationships" tab.
    10. Click on "Edit trust relationship"
    11. Replace the prepopulated text with the following, and replace the "{AWS_ACCOUNT_NUMBER}" with your AWS account id.  If 
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com",
        "AWS": "arn:aws:iam::{AWS_ACCOUNT_NUMBER}:role/Cloud9Instance"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
    12. Click "Update Trust Policy"
7. Setup AWS credentials for the Cloud9 environment
    1. Open the EC2 Console - https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home:
    2. Click "Instances"
    3. Click the checkbox for the instance name beginning with "aws-cloud9-SecHubWorkshop"
    4. Click the "Actions" button, then in the menu popup, click "Instance Settings", then "Attach/Replace IAM Role".
    5. In the "IAM role" box, select the "Cloud9Instance" role
    6. Click Apply.
    7. Open https://us-east-1.console.aws.amazon.com/cloud9/home?region=us-east-1
    8. In the box for "SecHubWorkshop", click "Open IDE"
    9. Within the Cloud9 browser tab, Click "File->New"
    10. Paste in the following text replacing "{AWS_ACCOUNT_NUMBER}" with your AWS account number
```
[profile cc]
region = us-east-1
role_arn = arn:aws:iam::{AWS_ACCOUNT_NUMBER}:role/CloudCustodian
credential_source = Ec2InstanceMetadata
role_session_name = cloudcustodian-via-cli
```
    11. Click File->Save
    12. In Folder, enter "~/.aws".
    13. In Filename, enter "config"
    14. Click "Save"
    15. Test the AWS Credentials by finding the terminal session at the bottom which starts with "bash", then enter "aws s3 ls --profile cc"
    16. If you get AccessDenied, then review the trust policy and IAMProfile associated with the instance as you need to have working credentials for CloudCustodian to work.
8. Install Cloud Custodian
    1. To install Cloud Custodian, just run the following in the bash terminal window of Cloud9:
```
python3 -m venv custodian
source custodian/bin/activate
pip install c7n
```

## Module 2 - Security Hub Custom Actions - Human initiated automation

## Module 3 - Automated Remediations - GuardDuty Event on EC2 Instance

## Module 4 - Automated Remediations - GuardDuty Event on IAMUser

## Module 5 - Automated Remediations - GuardDuty Event on EC2 Instance - Isolate rather than Stop



