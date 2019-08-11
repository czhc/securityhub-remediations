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
7. If any of your existing ec2 instances have their tag:Name=RemediationTestTarget then please rename them as instances with this value will be the target for actions during this workshop

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
            "Action": [
                "guardduty:CreateSampleFindings",
                "ec2:RunInstances",
                "ec2:StartInstance",
                "ec2:DescribeInstances",
                "ec2:AssociateIamInstanceProfile"
            ],
            "Resource": "*"
        },
        {
            "Sid": "PassRole",
            "Effect": "Allow",
            "Action": [
                "iam:PassRole"
            ],
            "Resource": [
                "arn:aws:iam::369510138361:role/Cloud9Instance"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameters"
            ],
            "Resource": "arn:aws:ssm:*:*:parameter/aws/service/ami-*"
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
```json
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
                "logs:CreateLogGroup",
                "logs:PutLogEvents"
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
```json
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
9. Setup an EC2 instance that will be the target of remediation actions 
```
aws ec2 run-instances --image-id $(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-minimal-hvm-x86_64-ebs --region us-east-1 --query 'Parameters[0].[Value]' --output text) --instance-type t2.micro --region us-east-1 --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=RemediationTestTarget}]' 
```
10. Test first Cloud Custodian Policy, which reports that the instance created in the previous step has a vulnerability
    1. Run the following:
```
custodian run -s /tmp --profile cc -c ~/environment/securityhub-remediations/module1/force-vulnerability-finding.yml
```
    2. You should expect to see 2 output lines, one containing "count:1" and another containing "resources:1", similar to the following, and if you get anything else, please troubleshoot and if can't figure out the problem, please ask for help before proceeding to next module.
```
2019-08-11 16:33:57,326: custodian.policy:INFO policy:ec2-force-vulnerabilities resource:ec2 region:us-east-1 count:1 time:0.00
2019-08-11 16:33:57,787: custodian.policy:INFO policy:ec2-force-vulnerabilities action:instancefinding resources:1 execution_time:0.46
```

## Module 2 - Security Hub Custom Actions - Human initiated automation
1. Run the following:
```
custodian run -s /tmp --profile cc -c ~/environment/securityhub-remediations/module2/ec2-sechub-custom-action.yml
```
2. You should see a single output line containing "custodian.policy:INFO Provisioning policy lambda DenySnapStop". Note that the string after 'Provisioning policy lambda" matches the poicy name contained within the file which was the last parameter of the previous step.  The name of the generated lambda will be composed of that policy name prefixed with "custodian-"
3. Open the Security Hub Console and click on Findings, or click https://console.aws.amazon.com/securityhub/home?region=us-east-1#/findings?search=RecordState%3D%255Coperator%255C%253AEQUALS%255C%253AACTIVE 
4. You should see a row where "Title=ec2-force-vulnerabilities", if not then in the Findings search box, type Title, under the pop-up Filters click on Title, then in the new popup, enter "ec2-force-vulnerabilities" then click Apply
5. Click the checkbox for the finding (There should only be one at this point, but checkbox the first (most recently updated)
6. Click "Actions" then in the popup click on "Ec2 DenySnapStop"
7. You should observe a green notification at top of page saying "Successfully send findings to Amazon CloudwatchEvents" and sometime in the future will include the action name once they implement my PFR.
8. Review the Cloudwatch log of the Lambda which got invoked.  LogGroupNames are composed of the prefix "/aws/lambda/custodian-" followed by the policy name. Lines with "ERROR" indicate something is wrong.  You should see at least a line containing "invoking action:" for each action in the policy.
9. Optional, you can use the AWS Console and/or cli to confirm that the instance named "RemediationTestTarget" has really be stopped, snapshotted, and the IAM Instance Profile dissassociated.
10. Now run the following commands to start the instance, and associate the InstanceProfile so the instance is ready for the next module.
```
aws ec2 start-instances --instance-ids $(aws ec2 describe-instances --filters "Name=tag:Name,Values=RemediationTestTarget" --query Reservations[*].Instances[*].[InstanceId] --output text)
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=Cloud9Instance --instance-id $(aws ec2 describe-instances --filters "Name=tag:Name,Values=RemediationTestTarget" --query Reservations[*].Instances[*].[InstanceId] --output text)
```

## Module 3 - Automated Remediations - GuardDuty Event on EC2 Instance

## Module 4 - Automated Remediations - GuardDuty Event on IAMUser

## Module 5 - Automated Remediations - GuardDuty Event on EC2 Instance - Isolate rather than Stop



