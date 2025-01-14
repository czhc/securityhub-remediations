# securityhub-remediations
Workshop for implementing remediations using AWS Security Hub and Cloud Custodian

# Overview
In this workshop you will learn how to implement automated remediations of findings submitted to [Security Hub](https://aws.amazon.com/security-hub/) leveraging an open source tool named [Cloud Custodian](https://cloudcustodian.io/), with no prior knowledge of either is required.  

However, this workshop is not intended to to provide a complete introduction to writing polices for Cloud Custodian, for that please refer to the [Getting Started documentation](https://cloudcustodian.io/docs/aws/gettingstarted.html) or alternatively the [introductory presentation on Cloud Custodian](https://www.socallinuxexpo.org/sites/default/files/presentations/CloudCustodian%40Scale17x.pdf)

Your feedback is highly desired, please [submit a new issue](https://github.com/FireballDWF/securityhub-remediations/issues/new) if you run into any problems, even if you figure it out yourself, please report the problem so we can attempt to make this workshop as error proof as possible. 

* Level: Intermediate
* Duration: 1:00 - 2:00 hours
* CSF Functions: Detect, Respond
* CAF Components: Detective, Responsive

# Prerequisites

1.  You will need an [AWS account](https://aws.amazon.com/account/) for this workshop and administrative credentials, with console and [aws cli access](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html). 
2.  We suggest you use a new/clean account, or at least one in which can tolerate the termination, stopping, and/or deleting of resources, and doesn't already have automated remediations of security groups or EC2 instances with public access, or removal of IAM users.
3.  You will incur charges for the AWS resources used in this workshop. The charges for some of the resources may be covered through the [AWS Free Tier](https://aws.amazon.com/free/). The demo uses free tier choices wherever possible.
4.  You must run this workshop in a [region supported by AWS Security Hub](https://docs.aws.amazon.com/general/latest/gr/rande.html#sechub_region).  We recommended using the us-east-1 region.
5.  You must run this workshop in a region support by [AWS Cloud9](https://docs.aws.amazon.com/general/latest/gr/rande.html#cloud9_region), 
or be comfortable setting up a docker environment with aws credentials in the host env.
6.  If any of your existing ec2 instances have their tag:Name=RemediationTestTarget then please rename them as instances with this value will be the target for actions during this workshop
7.  If you choose to install into an existing, rather than enabling the option to create a new VPC, be aware of the following:
    1.  VPC needs to have a connectivity to the following service's endpoints, such as thru a IGW or TGW:
        1.  EC2 - could also be via VPCEndpoints
        2.  Lambda - could also be via VPCEndpoints
        3.  SSM (Systems Manager) - could also be via VPCEndpoints
        4.  SecurityHub - VPC Endpoints not supported AsOf last update to this doc
        5.  GuardDuty - VPC Endpoints not supported AsOf last update to this doc
        6.  Cloud9 - Only needed if the cloudformation parameter CreateCloud9Instance is set to True.  
        7.  Config (only in module 4) - could also be via VPCEndpoints
        8.  STS (only in module 4) - could also be via VPCEndpoints
        9.  Cloudwatch Logs and Events - could also be via VPCEndpoints
    2. If using an S3 VPCEndpoint, access needs to be provided to a [list of public s3 buckets owned by Amazon for use with SSM](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-instance-profile.html#instance-profile-custom-s3-policy).  
8. Cloudtrail need to be enabled in the region
9. An understanding of [Cloudwatch Events](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html) is useful, but not required, for this Workshop

# Modules

1. Module 1 - Environment Build and Configuration
2. Module 2 - Security Hub Custom Actions - Human initiated automation
3. Module 3 - Automated Remediations - GuardDuty DNS Event on EC2 Instance
4. Module 4 - Automated Remediations - Vulnerability Event on EC2 Instance with Very Risky Configuration
5. Module 5 - Automated Remediations - GuardDuty Event on IAMUser


## Module 1 - Environment Build and Configuration

1.  If you don't already have SecurityHub enabled in the account and region you plan on using, then run:
```
aws securityhub enable-security-hub
```
2.  You need to have GuardDuty enabled on the account for module 2 and 5 to work, if not yet then either run the following command or follow the [steps to enable on the console](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_settingup.html#guardduty_enable-gd)
```
aws guardduty create-detector --enable
```
3.  Run "wget https://github.com/FireballDWF/securityhub-remediations/blob/master/module1/securityhub-remediations-workshop.yml" which downloads a copy of the cloudformation template used in the next step.  Alternatively if you don't have a wget, you could use curl or your browser to download the same url.
4.  Use the AWS Console to launch a cloudformation stack using the template downloaded in the previous step. If if you launch from the cli, the role must match your console role otherwise you won't be able to see the Cloud9 Environment IDE.
5.  Assign an IAM Instance Profile to the ec2 instance for the Cloud9 environment
```
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=SecurityHubRemediationWorkshopCli --instance-id $(aws ec2 describe-instances --filters Name=tag:Name,Values="aws-cloud9-SecurityHubWorkshop*" Name=instance-state-name,Values=running --query Reservations[*].Instances[*].[InstanceId] --output text)
```
6.  Verify that the output from the prior command contained "State: associating"
7.  Find the terminal session at the bottom which starts with "bash".  You will use it for the remainder of the Workshop anytime their is a command line to be run. Use it to run the following commands so that you have a copy of the workshop files on your Cloud9 instance and have a directory for output from Cloud Custodian: 
```
git clone https://github.com/FireballDWF/securityhub-remediations.git
cd securityhub-remediations
mkdir output
```
7.  Run the following in the same Cloud9 terminal to setup an environment varible required by upcoming commands:
```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```
8.  The following environment variable sets the default region to use, change it from us-east-1 if desired, then run it:
```
export AWS_DEFAULT_REGION=us-east-1
```
9.  Configure credentials for deploying cloud custodian by running the following, and change the region if not targeting us-east-1:
```
echo -e "[profile cc]\n\
region = ${AWS_DEFAULT_REGION}\n\
role_arn = arn:aws:iam::${AWS_ACCOUNT_ID}:role/CloudCustodianCli\n\
credential_source = Ec2InstanceMetadata\n\
role_session_name = cloudcustodian-via-cli\n"  >>~/.aws/config 
```
10.  Test the ability to use the new credentials profile named cc
```
aws s3 ls --profile cc
```
11.  If you get AccessDenied, then troubleshoot the cli credentials issue as they are required by rest of the modules.  If you get "Invalid endpoint: https://s3..amazonaws.com" then you missed the step setting AWS_DEFAULT_REGION.  If you get "An error occurred (InvalidClientTokenId) when calling the AssumeRole operation: The security token included in the request is invalid." then you likely did not perform the step to run the command which starts with "aws ec2 associate-iam-instance-profile"
12.  Install Cloud Custodian
    1. To install Cloud Custodian, just run the following in the bash terminal window of Cloud9:
```
export SECHUBWORKSHOP_CONTAINER=cloudcustodian/c7n
docker pull ${SECHUBWORKSHOP_CONTAINER} 
```
13.  Test first Cloud Custodian Policy, which reports that the ec2 instance created in the cloudformation has a vulnerability
    1. Run the following:
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module1/force-vulnerability-finding.yml  
```

    You should expect to see 2 output lines, one containing "count:1" and another containing "resources:1", similar to the following output.  If you get an error on "batch-import-findings" then it means you have not enabled SecurityHub in the account and region.

```
2019-08-11 16:33:57,326: custodian.policy:INFO policy:ec2-force-vulnerabilities resource:ec2 region:us-east-1 count:1 time:0.00
2019-08-11 16:33:57,787: custodian.policy:INFO policy:ec2-force-vulnerabilities action:instancefinding resources:1 execution_time:0.46
```

    Here is a breakdown of the command you just ran:
    
| Command line Component | Explaination |
| --------- | ------------ |
| docker run | [Run](https://docs.docker.com/engine/reference/run/) a [docker](https://docs.docker.com/engine/reference/commandline/cli/) container |
| -it | [interactive/foreground mode](https://docs.docker.com/engine/reference/run/#foreground) |
| --rm | [clean up docker container when container exits](https://docs.docker.com/engine/reference/run/#clean-up----rm) |
| --group-add 501 | configs the container process to execute being a member of unix group 501 so that it's allowed to write to the output directory
| -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw | [maps a directory](https://docs.docker.com/engine/reference/run/#volume-shared-filesystems) for the output of Cloud Custodian between the container host and the container instance. volume is read/write enabled |
| -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro|map the files for the workshop into the container so the cloud custodian policies are available insider the container. volume is mapped in ReadOnly mode |
| -v /home/ec2-user/.aws:/home/custodian/.aws:ro | maps the [aws cli configuration files](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) into the container in read-only mode.  Cloud Custodian uses the same configuration files, as both use the [boto3 Python SDK](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) |
| ${SECHUBWORKSHOP_CONTAINER} | evaluates to cloudcustodian/c7n which is the docker container image which is downloaded from https://hub.docker.com/r/cloudcustodian/c7n |
| run | instructs Cloud Custodian to run a policy. This is the first part of the command line which is passed to CloudCustodian |
| --cache-period 0 | disables cloud custodian's caching of api call results |
| -s /home/custodian/output | specifies where log and resource data is placed | 
| --profile cc | specifies which AWS credentials profile to use | 
| -c /home/custodian/securityhub-remediations/module1/force-vulnerability-finding.yml | specifies the actual policy to run | 

## Module 2 - Security Hub Custom Actions - Human initiated automation
1. Run the following:
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module2/ec2-sechub-custom-actions.yml
```
2. You should see lines of output line like the following:
```
2019-09-01 19:20:29,021: custodian.policy:INFO Provisioning policy lambda DenySnapStop
2019-09-01 19:20:30,885: custodian.policy:INFO Provisioning policy lambda DisableKey
2019-09-01 19:20:30,885: custodian.policy:INFO Provisioning policy lambda Delete
2019-09-01 19:20:30,885: custodian.policy:INFO Provisioning policy lambda PostOpsItem
2019-09-01 19:20:30,885: custodian.policy:INFO Provisioning policy lambda RemPA
2019-09-01 19:20:32,222: custodian.serverless:INFO Publishing custodian policy lambda function custodian-DenySnapStop
2019-09-01 19:20:32,222: custodian.serverless:INFO Publishing custodian policy lambda function custodian-DisableKey
2019-09-01 19:20:32,222: custodian.serverless:INFO Publishing custodian policy lambda function custodian-Delete
2019-09-01 19:20:32,222: custodian.serverless:INFO Publishing custodian policy lambda function custodian-PostOpsItem
2019-09-01 19:20:32,222: custodian.serverless:INFO Publishing custodian policy lambda function custodian-RemPA
```
3. Note that the string after 'Provisioning policy lambda" matches the policy names contained within the ec2-sechub-custom-actions.yml file from the last docker command.  The namse of the generated lambda will be composed of that policy names prefixed with "custodian-".  Cloudwatch logs are generated following standard naming convention, /aws/lamabda/custodian-$(PolicyName)
3. Open the Security Hub Console and click on Findings, or click https://console.aws.amazon.com/securityhub/home?region=us-east-1#/findings?search=RecordState%3D%255Coperator%255C%253AEQUALS%255C%253AACTIVE 
4. You should see a row where "Title=ec2-force-vulnerabilities", if not then in the Findings search box, type Title, under the pop-up Filters click on Title, then in the new popup, enter "ec2-force-vulnerabilities" then click Apply
5. Click the checkbox for the finding (There should only be one at this point, but checkbox the first (most recently updated)
6. Click "Actions" then in the popup click on "Ec2 DenySnapStop"
7. You should observe a green notification at top of page saying "Successfully send findings to Amazon CloudwatchEvents" and sometime in the future will include the action name once they implement my PFR.
8. Review the Cloudwatch log of the Lambda which got invoked.  LogGroupNames are composed of the prefix "/aws/lambda/custodian-" followed by the policy name. Lines with "ERROR" indicate something is wrong.  You should see at least a line containing "invoking action:" for each action in the policy.
9. Optional, you can use the AWS Console and/or cli to confirm that the instance named "RemediationTestTarget" has really be stopped, snapshotted, and the IAM Instance Profile dissassociated.
10. Now run the following command to reassociate the InstanceProfile so the instance is ready for the next module.
```
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=SecurityHubRemediationWorkshopCli --instance-id $(aws ec2 describe-instances --filters "Name=tag:Name,Values=RemediationTestTarget Name=instance-state-name,Values=stopped" --query Reservations[*].Instances[*].[InstanceId] --output text)
```
11. Now run the following command to start the instance so the instance is ready for the next module.
```
aws ec2 start-instances --instance-ids $(aws ec2 describe-instances --filters "Name=tag:Name,Values=RemediationTestTarget" --query Reservations[*].Instances[*].[InstanceId] --output text)
```
## Module 3 - Automated Remediations - GuardDuty DNS Event on EC2 Instance
1.  Run the following command which runs a policy named [ec2-sechub-remediate-severity-with-findings](https://github.com/FireballDWF/securityhub-remediations/blob/master/module3/ec2-sechub-remediate-severity-with-findings.yml) which instructs Cloud Custodian to dynamically generate and deploy a lambda, which will be invoked when [SecurityHub generates a Cloudwatch Event](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cloudwatch-events.html) when sent a finding. In this module, the finding will be triggered when GuardDuty generates a finding, and the severity of the is greater than or equal to 31, and the EC2 instance has any vulnerability previously reported to SecurityHub
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module3/ec2-sechub-remediate-severity-with-findings.yml
```
2.  Next run the following command which leverages [Systems Manager's Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html) to run an "nslookup" command on the ec2 instance tag:Name RemediationTestTarget where it's looking up a dns name which [GuardDuty will detect as Command and Control activity](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_backdoor.html#backdoor7)
```
aws ssm send-command --document-name AWS-RunShellScript --parameters commands=["nslookup guarddutyc2activityb.com"] --targets "Key=tag:Name,Values=RemediationTestTarget" --comment "Force GuardDutyFinding" --cloud-watch-output-config CloudWatchLogGroupName=/aws/ssm/AWS-RunShellScript,CloudWatchOutputEnabled=true --profile cc
```
3.  As it can take a long time (more than 20 minutes often around 2 hours) for GuardDuty to generate a DNS based finding, please proceed to the next module, then come back to the next review step later.
4.  Review the Logs via https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/lambda/custodian-ec2-sechub-remediate-severity-with-findings;streamFilter=typeLogStreamPrefix 

## Module 4 - Automated Remediations - Vulnerability Event on EC2 Instance with Very Risky Configuration
1.  Run the following command, which invokes Cloud Custodian to run a policy named [ec2-public-ingress-hubfinding](https://github.com/FireballDWF/securityhub-remediations/blob/master/module4/ec2-public-ingress-hubfinding.yml) which filters for a high risk configuation (Details TODO).
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module4/ec2-public-ingress-hubfinding.yml
```
2.  Run the following command to trigger an finding event in Security Hub on the with the resource being the RemediationTestTarget instance. 
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module1/force-vulnerability-finding.yml  
```
3.  Review the Logs via https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/lambda/custodian-ec2-public-ingress-hubfinding;streamFilter=typeLogStreamPrefix 
4.  Review module4/ec2-public-ingress.yml observing that the lack of a "mode" section means it can be run anytime to find the risky configuration without requiring a vulnerability event.
5.  Now run the following command to re-associate the InstanceProfile so the instance is ready for the next module.
```
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=SecurityHubRemediationWorkshopCli --instance-id $(aws ec2 describe-instances --filters "Name=tag:Name,Values=RemediationTestTarget" --query Reservations[*].Instances[*].[InstanceId] --output text)
```
6.  Future Addition: Similar policy as a config rule
  
## Module 5 - Automated Remediations - GuardDuty Event on IAMUser
1. Run the following command:
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module5/iam-user-hubfinding-remediate-disable.yml
```
2.  Verify that the previous command resulted in output containing "Provisioning policy lambda iam-user-hubfinding-remediate-disable"
3.  Next archive any existing sample GuardDuty Findings for the IAM User named GeneratedFindingUserName.  While this is not nessesary when the create-sample-findings command (which is run later in this module) has never been run before, it won't harm anything to run.  And if it's not run and the sample finding has already been generated, then the cloudwatch event needed for this module to function never gets triggered, so we're running it to eliminate sources of potential error.  And if you want to rerun this module, you need to run this command.
```
aws guardduty archive-findings \
    --detector-id \
        $(aws guardduty list-detectors --query DetectorIds --output text) \
    --finding-ids \
        $(aws guardduty list-findings \
            --detector-id \
                $(aws guardduty list-detectors --query DetectorIds --output text) \
            --finding-criteria '{"Criterion": {"service.archived": {"Eq":["false"]},"resource.accessKeyDetails.userName": {"Eq":["GeneratedFindingUserName"]}}}' --query 'FindingIds[0]' --output text)
```
4.  Archive any existing findings of this type in Security Hub, to be on the safe side.
```
aws securityhub update-findings --record-state ARCHIVED --filters '{"ResourceAwsIamAccessKeyUserName":[{"Value": "GeneratedFindingUserName","Comparison":"EQUALS"}]}' 
```
5.  Run the following command to validate that the Access Keys status is currently active:
```
aws iam list-access-keys --user-name GeneratedFindingUserName
```
6. Run the following command, which creates a sample finding in GuardDuty, which automatically get imported into SecurityHub, which is an finding type ['UnauthorizedAccess:IAMUser/MaliciousIPCaller'](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_unauthorized.html#unauthorized5) on an IAMUser named GeneratedFindingUserName, which was created by cloudformation script in module 1.
```
aws guardduty create-sample-findings --detector-id `aws guardduty list-detectors --query DetectorIds --output text` --finding-types 'UnauthorizedAccess:IAMUser/MaliciousIPCaller'
```
7.  First, validate that Guard Duty generated the sample finding by going to the Guard Duty Console and look for the finding type "UnauthorizedAccess:IAMUser/MaliciousIPCaller"
8.  While it can take a few minutes for the finding to show up in the Security Hub console, look for the finding Title = "API GeneratedFindingAPIName was invoked from a known malicious IP address." or use the following URL to search: https://console.aws.amazon.com/securityhub/home?region=us-east-1#/findings?search=RecordState%3D%255Coperator%255C%253AEQUALS%255C%253AACTIVE%26ResourceAwsIamAccessKeyUserName%3D%255Coperator%255C%253AEQUALS%255C%253AGeneratedFindingUserName
9.  Look within the CloudWatch Logs, remember the LogGroup pattern of /aws/lambda/custodian-$(name of the cloud custodian policy).  You are looking a lines containing "policy:iam-user-hubfinding-remediate-disable invoking action:userremoveaccesskey resources:1" followed by a line containing "metric:ApiCalls Count:2 policy:iam-user-hubfinding-remediate-disable restype:iam-user", if they don't appear will need to troubleshoot using the logs.
10.  Validate that the Access Keys were actually removed from the user by running the following command:
```
aws iam list-access-keys --user-name GeneratedFindingUserName
```
10. Evaluate the output by looking for "Status"="Inactive"
11. Optional Steps: One of the other Custom Actions which got deployed to Security Hub is named "Iam-User DisableKey". If you would like to test it, run the following command to reactivate the AccessKey, then repeat the step (earlier in this module) for generating a sample finding with GuardDuty, wait for the finding event to be generated and show up in SecurityHub, then view the finding in SecurityHub, check the box for the finding, then select the custom action named "Iam-User DisableKey", then repeat the "list-access-key" step to confirm the inactive status of the keys.
```
aws iam update-access-key --user-name GeneratedFindingUserName --status Active --access-key-id $(aws iam list-access-keys --user-name GeneratedFindingUserName --query AccessKeyMetadata[0].AccessKeyId --output text) --profile cc
``` 

## Module 6 - Remediate an Public EBS-Snapshot 

This module will show how to setup an automated detection of a EBS Snaspshot that has been made public, with a finding submitted to Security Hub, then use Security Hub Custom action to delete the snapshot.  Then we'll fully automate the remediation by changing the detection policy to perform the delete while still providing notification.  
1. Start by reviewing the file "post-ebs-snapshot-public.yml" using the Cloud9 IDE. Observe that the only action type is "post-finding". Also Observe that the following two lines instruct Cloud Custodian to deploy a CloudWatch rule which triggers on a regular schedule, configured to be every 5 minutes.
```
      type: periodic
      schedule: "rate(5 minutes)"
```
2. Then deploy the automated detection policy by running the following command:
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module6/post-ebs-snapshot-public.yml
```
3. Next we need to create a new EBS volume, however no data will be stored in it, it will not be attached anywhere.  Note how the VolumeId is saved for the next step.
```
export WorkshopVolumeId=$(aws ec2 create-volume --availability-zone $(aws ec2 describe-availability-zones --query AvailabilityZones[0].ZoneName --output text) --size 1 --query VolumeId --output text)
```

4. Then we Snapshot the volume just created, also saving the SnapshotId for a future step.
```
export WorkshopSnapshotId=$(aws ec2 create-snapshot --volume-id $WorkshopVolumeId --query SnapshotId --output text)
```

5. This step makes the snapshot public.
```
aws ec2 modify-snapshot-attribute --snapshot-id $WorkshopSnapshotId --attribute createVolumePermission --operation-type add --group-names all
```

6. Now navigate to the Findings part of Security Hub's console.  Look for a finding who's title is the same as the policy name from step 1 then click it's checkbox.  If you don't see it, you may need to wait up to 5 minutes (remember the schedule based CloudWatch Rule from Step 1) for it to appear after refreshing the browser.
7. Click the dropdown for Actions then select "Ebs-Snapshot Delete" (This custom action is one of the ones deployed in Module 2).
8. Confirm the snapshot got deleted by running:
```
aws ec2 describe-snapshots --snapshot-ids $WorkshopSnapshotId
```
then confirming the response is similar to:
```
An error occurred (InvalidSnapshot.NotFound) when calling the DescribeSnapshots operation: The snapshot 'snap-0643b6dcd0a6f01f0' does not exist.
```

9. Now edit the file "post-ebs-snapshot-public.yml" by adding an action to delete the snapshot at time of initial detection, which transform the policy from a detective control to a remediation control.  Append the following line to the file, where the dash should be in column 5.
```
    - type: delete
```
10. Save the file in the IDE.
11. Run the following command which redeploys the policy:
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module6/post-ebs-snapshot-public.yml
```
12. Run the following three commands:
```
export WorkshopSnapshotId=$(aws ec2 create-snapshot --volume-id $WorkshopVolumeId --query SnapshotId --output text)
aws ec2 modify-snapshot-attribute --snapshot-id $WorkshopSnapshotId --attribute createVolumePermission --operation-type add --group-names all
aws ec2 describe-snapshots --snapshot-ids $WorkshopSnapshotId
```
13. Repeat running the last command, the describe-snapshots one, until either the InvalidSnapshot.NotFound error is received (which means success, move on the the next step) or if after 5 minutes it still has not been deleted, the most likely cause is Step 9 did not get done correctly, review the CloudWatch logs for the policy, or the file did not get saved (view the file from the terminal to see if it has the type: delete line), or the policy did not get redeployed in step 11.
14. The remainder of this module shows you how to customize the automated remediation by adding an exception filter then provides information on more advanced filters.  If you have a use case for publicly sharing a snapshot, a change to the policy could be made to filter for only those snapshots which do not have a specific value for a specific tag. In this example we use the Tag key of "PublicIntent" and Tag value of "True".  Start by updating the filter section of the policy by inserting the following lines at line 13 or 16, as long as it's within the filter section and doesn't overlap the existing filter, with the dash character at column 13.  The way to read the filter match the condition of not-equal when testing for Tag key of value. Thus resources which have this tag value will get filtered out aka excluded, thus the actions will not get invoked on those filtered out resources.
```
      - type: value
        op: not-equal
        key: "tag:PublicIntent"
        value: "True"
```
15. Save the file.  
16. Deploy the updated policy
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output --profile cc -c /home/custodian/securityhub-remediations/module6/post-ebs-snapshot-public.yml
```
17. Run the following command noticing that this time it's created with a Tag key and value which matches the filter specified.
```
export WorkshopSnapshotId=$(aws ec2 create-snapshot --volume-id $WorkshopVolumeId --query SnapshotId --output text --tag-specifications 'ResourceType=snapshot,Tags=[{Key=PublicIntent,Value=True}]')
```
18. Run the following to make the snapshot public:
```
aws ec2 modify-snapshot-attribute --snapshot-id $WorkshopSnapshotId --attribute createVolumePermission --operation-type add --group-names all
```
19. Run the following which waits for 5 minutes to then show the snapshot did not get deleted, a result of the exception filter.
```
sleep 300 ; aws ec2 describe-snapshots --snapshot-ids $WorkshopSnapshotId 
```
20. For good hygene, delete the public snapshot manually.
```
aws ec2 delete-snapshots --snapshot-ids $WorkshopSnapshotId
```
21. To learn more about the types of filters that can be added to any Cloud Custodian Policy, click [generic filters](https://cloudcustodian.io/docs/filters.html).  
22. To learn about the filters that can be applied to EBS Snapshots, run the following:
```
docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.filters
```
23. To learn more about the attributes of a specific filter, append the filters name to the command, like the following example for the skip-ami-snapshots filter
```
docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.filters.skip-ami-snapshots
```
26. To learn about what AWS resource types are supported by Cloud Custodian, run the following:
```
docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema aws
```
27. The actions supported by a specific resource can be viewed by using the schema command with the parameter of <resource_type>.actions, like in the following:
```
docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.actions
```
28. And just like filters, the attributes for a given action can be viewed by running the schema command specifing the <resource_type>.actions.<action_name>, like the following:
```
docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.actions.post-finding
```
 


## Module X - Cleanup
1.  For maximum cleanup:
    1. delete the cloudformation stack 
    2. disable SecurityHub, which is really not recommended.
    3. TODO: Details about CloudCustodian lambda, cloudwatch events and logs
2.  If you did not choose to execute the delete the cloudformation stack step, you should at least terminate the EC2 instance used as a test target by runnning the following:
```
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances --filters "Name=tag:Name,Values=RemediationTestTarget" --query Reservations[*].Instances[*].[InstanceId] --output text)
```
