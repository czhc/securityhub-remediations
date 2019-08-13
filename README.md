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
8. Resources will be created in the default vpc.  If you don't have a default vpc, you will need to modify the commands to specify the vpc you want to use.
9. A git client to download the workshop files
10. If your account already has automated remediations which respond to security groups with public ingress, ec2 instances with public ips, please use an account which doesn't have those remediations, or temporary turn them off, otherwise race conditions will lead to results which don't match what this workshop describes.

# Modules

1. Module 1 - Environment Build and Configuration
2. Module 2 - Security Hub Custom Actions - Human initiated automation
3. Module 3 - Automated Remediations - GuardDuty DNS Event on EC2 Instance
4. Module 4 - Automated Remediations - Vulnerability Event on EC2 Instance with Very Risky Configuration
5. Module 5 - Automated Remediations - GuardDuty Event on IAMUser


## Module 1 - Environment Build and Configuration
0. Run "git clone https://github.com/FireballDWF/securityhub-remediations.git && cd securityhub-remediations"
1. Enable Security Hub (if not already enabled - https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-settingup.html#securityhub-enable
2. Launch cloudformation to setup the environment
    1. Use the console to launch a cloudformation stack using the template module1/securityhub-remediations-workshop.yml as if you launch from the cli, the role must match your console role otherwise you won't be able to see the Cloud9 Environment IDE.
3. Setup AWS credentials for the Cloud9 environment
    1. Open the EC2 Console - https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Home:
    2. Click "Instances"
    3. Click the checkbox for the instance name beginning with "aws-cloud9-SecHubWorkshop"
    4. Click the "Actions" button, then in the menu popup, click "Instance Settings", then "Attach/Replace IAM Role".
    5. In the "IAM role" box, select the "Cloud9Instance" role
    6. Click Apply.
    7. Open https://us-east-1.console.aws.amazon.com/cloud9/home?region=us-east-1
    8. In the box for "SecHubWorkshop", click "Open IDE"
    9. Find the terminal session at the bottom which starts with "bash" and use it to run: "git clone https://github.com/FireballDWF/securityhub-remediations.git && cd securityhub-remediations" so that you have a copy of the workshop files on your Cloud9 instance
    10. Within the Cloud9 browser tab, open the file securityhub-remediations/module1/config
    11. Replace "{AWS_ACCOUNT_NUMBER}" with your AWS account number
    12. Click File->Save As
    13. In Folder, enter "~/.aws"
    14. Click "Save"
    15. Test the AWS Credentials by going to the IDE's terminal window then enter "aws s3 ls --profile cc"
    16. If you get AccessDenied, then review the edits you made to ~/.aws/config and step 5 as you need to have working credentials for CloudCustodian to work.
4. Install Cloud Custodian
    1. To install Cloud Custodian, just run the following in the bash terminal window of Cloud9:
```
python3 -m venv custodian
source custodian/bin/activate
pip install c7n
```
5. Test first Cloud Custodian Policy, which reports that the instance created in the previous step has a vulnerability
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

## Module 3 - Automated Remediations - GuardDuty DNS Event on EC2 Instance
1. Run the following commands, the first deploys a Cloud Custodian policy which will be triggered when there are GuardDuty findings there are equal to or greater than medium and the EC2 instance has any vulnerability reported to SecurityHub, and the 2nd command simulates an GuardDuty finding.
```
custodian run -s /tmp --profile cc -c ~/environment/securityhub-remediations/module3/ec2-sechub-remediate-severity-with-findings.yml
aws ssm send-command --document-name AWS-RunShellScript --parameters commands=["nslookup guarddutyc2activityb.com"] --targets "Key=tag:Name,Values=RemediationTestTarget" --comment "Force GuardDutyFinding" --cloud-watch-output-config CloudWatchLogGroupName=/aws/ssm/AWS-RunShellScript,CloudWatchOutputEnabled=true
```
2.  Review the Logs via https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/lambda/custodian-ec2-sechub-remediate-severity-with-findings;streamFilter=typeLogStreamPrefix 

## Module 4 - Automated Remediations - Vulnerability Event on EC2 Instance with Very Risky Configuration
1. Run the following commands:
```
custodian run -s /tmp --profile cc -c ~/environment/securityhub-remediations/module4/ec2-public-ingress-hubfinding.yml
custodian run -s /tmp --profile cc -c ~/environment/securityhub-remediations/module1/force-vulnerability-finding.yml
```
2. Review the Logs via https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/lambda/custodian-ec2-public-ingress-hubfinding;streamFilter=typeLogStreamPrefix 
3. Review module4/ec2-public-ingress.yml observing that the lack of a "mode" section means it can be run anytime to find the risky configuration without requiring a vulnerability event.
4. Now run the following command to re-associate the InstanceProfile so the instance is ready for the next module.
```
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=Cloud9Instance --instance-id $(aws ec2 describe-instances --filters "Name=tag:Name,Values=RemediationTestTarget" --query Reservations[*].Instances[*].[InstanceId] --output text)
```

## Module 5 - Automated Remediations - GuardDuty Event on IAMUser
1. Run the following commands:
```
custodian run -s /tmp --profile cc -c ~/environment/securityhub-remediations/module5/iiam-user-hubfinding-remediate-disable.yml
aws guardduty create-sample-findings --detector-id `aws guardduty list-detectors --profile cc --query DetectorIds --output text` --finding-types 'UnauthorizedAccess:IAMUser/MaliciousIPCaller'
```

TODO: update to create iam-user named "GeneratedFindingUserName" with accesskeys 


