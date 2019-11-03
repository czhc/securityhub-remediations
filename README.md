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
1. Ability to navigate within the AWS Management Console to open specific Service consoles and features, including CloudWatch Logs and Lambda.
2. If you don't already have an understanding of [Cloudwatch Events](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html) please follow the link and read the content so you understand this fundamental technology.
3. If you don't already have an understanding of how Security Hub integrates with Cloudwatch Events, then suggested reading is [Automating AWS Security Hub with CloudWatch Events](https://docs.aws.amazon.com/en_pv/securityhub/latest/userguide/securityhub-cloudwatch-events.html#securityhub-cwe-configure). However it's not required as this workshop will walk you thru one specific method for how to deploy automated remediations with Security Hub.

# Modules

1. Module 1 - Environment Build and Configuration
2. Module 2 - Security Hub Custom Actions - Human initiated automation
3. Module 3 - Automated Remediations - GuardDuty DNS Event on EC2 Instance
4. Module 4 - Automated Remediations - Vulnerability Event on EC2 Instance with Very Risky Configuration
5. Module 5 - Automated Remediations - GuardDuty Event on IAMUser
6. Module 6 - Remediate an Public EBS-Snapshot 


## Module 1 - Environment Build and Configuration

Note: GuardDuty and SecurityHub have already been enabled in the account used for this workshop.
1.  First, from your Event Engine Team Role tab, click "Open Console" button within the Login Link section.
2.  Next open the Cloud9 IDE which provides the ability to review files and execute commands in a browser based terminal window.  One quick way is to type "Cloud9" then hit Enter in the "Find Services" textbox near the top of the Management Console.
3.  Now click the "Open IDE" button.
4.  In the bottom part of the browser tab which opens up, look for a tab with a label starting with "bash", where the window contents are "TeamRole:~/environment $".  This is the browser based terminal session you'll use for the rest of the workshop for any command line steps.  
5.  The next step is to get a copy of the files required for this workshop by cloning the workshop's github repo specifing the eventengine branch.
```
git clone --single-branch --branch eventengineenablement https://github.com/FireballDWF/securityhub-remediations.git
```
6.  Next step is to change into the directory created by the clone operation and then create a directory for the outpot of some commands to be written to.
```
cd securityhub-remediations
mkdir output

```
7.  Next step is to pull down the latest version of the Cloud Custodian docker container image.
```
export SECHUBWORKSHOP_CONTAINER=cloudcustodian/c7n
docker pull ${SECHUBWORKSHOP_CONTAINER} 
```
8.  This step tests the environment by invoking a Cloud Custodian Policy which reports that an ec2 instance has a vulnerability.
```
docker run -it --rm --cap-drop ALL --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module1/force-vulnerability-finding.yml  
```

You should expect to see 2 output lines, one containing "count:1" and another containing "resources:1", similar to the following output.  If you get an error on "batch-import-findings" then it means SecurityHub has not been enabled.  Example output is from us-east-1, however your results should indicate the region being used for the workshop event.

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
| --cap-drop ALL | Drop all Linux kernel capabilities as recommended in Rule #3 of the [Docker Security Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Docker_Security_Cheat_Sheet.md) |
| --group-add 501 | configs the container process to execute being a member of unix group 501 so that it's allowed to write to the output directory
| -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw | [maps a directory](https://docs.docker.com/engine/reference/run/#volume-shared-filesystems) for the output of Cloud Custodian between the container host and the container instance. volume is read/write enabled |
| -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro|map the files for the workshop into the container so the cloud custodian policies are available insider the container. volume is mapped in ReadOnly mode |
| -v /home/ec2-user/.aws:/home/custodian/.aws:ro | maps the [aws cli configuration files](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) into the container in read-only mode.  Cloud Custodian uses the same configuration files, as both use the [boto3 Python SDK](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) |
| ${SECHUBWORKSHOP_CONTAINER} | evaluates to cloudcustodian/c7n which is the docker container image which is downloaded from https://hub.docker.com/r/cloudcustodian/c7n |
| run | instructs Cloud Custodian to run a policy. This is the first part of the command line which is passed to CloudCustodian |
| --cache-period 0 | disables cloud custodian's caching of api call results |
| -s /home/custodian/output | specifies where log and resource data is placed | 
| -c /home/custodian/securityhub-remediations/module1/force-vulnerability-finding.yml | specifies the actual policy to run | 
9.  If you received the expected output lines, congratulations, you have successfully tested the environment setup by having Cloud Custodian submit a finding to Security Hub.  Proceed to the next module.

## Module 2 - Security Hub Custom Actions - Human initiated automation
Custom Actions in Security Hub are useful for analysts working with the Security Hub console who want to send a specific finding, or a small set of findings, to a response or remediation workflow.
The finding generated by the test in the first module will be used within this module to explore Custom Actions.  

1. Run the following:
```
docker run -it --rm --cap-drop ALL --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module2/ec2-sechub-custom-actions.yml
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
3. Note that the string after 'Provisioning policy lambda" matches the policy names contained within the ec2-sechub-custom-actions.yml file from the last docker command.  The names of the generated lambdas will be composed of that policy names prefixed with "custodian-".  Cloudwatch logs are generated following standard naming convention, /aws/lamabda/custodian-$(PolicyName)
4. Within the Management Console, navigate to the Security Hub service.
5. In the left hand navigation area, click on Findings.
6. You should see a row where the value of the Title column is "ec2-force-vulnerabilities", if not then in the Findings search box, type Title, under the pop-up Filters click on Title, then in the new popup, enter "ec2-force-vulnerabilities" then click Apply.
7. Click the checkbox (left hand side) for the finding.  
8. In the upper right, click "Actions" then in the popup click on "Ec2 DenySnapStop"
9. You should observe a green notification at top of page saying "Successfully send findings to Amazon CloudwatchEvents".  I've submitted a request to include the action name in that message.
10. Review the Cloudwatch log of the Lambda which got invoked.  Log Group names are composed of the prefix "/aws/lambda/custodian-" followed by the policy name, so in this case "aws/lambda/custodian-DenySnapStop". Within that log group, open the most recent Log stream.  Lines with "ERROR" indicate something is wrong, please let the event facilitor know if you see an ERROR.  You should see at least a line containing "invoking action:" for each action in the policy. 
11. Optional: Review the complete payload of the Cloudwatch event which is logged directly after a line (usually line #2) ending with the text "Processing event".
12. Optional, you can use the AWS Console and/or cli to confirm that the instance named "RemediationTestTarget" has really be stopped, snapshotted, and the IAM Instance Profile dissassociated.
13. Now run the following command to reassociate the InstanceProfile as it's needed for the next module.
```
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=SecurityHubRemediationWorkshopCli --instance-id $(aws ec2 describe-instances --filters Name=tag:Name,Values=RemediationTestTarget --query Reservations[*].Instances[*].[InstanceId] --output text)
```
14. Now run the following command to start the instance so the instance is ready for the next module.
```
aws ec2 start-instances --instance-ids $(aws ec2 describe-instances --filters Name=tag:Name,Values=RemediationTestTarget Name=instance-state-name,Values=stopped --query Reservations[*].Instances[*].[InstanceId] --output text)
```
If you get an error, the most likely reason is that the instance is still in the stopping state, wait 5-10 seconds then retry.
15.  Optional: Using the same finding from step 6 & 7, invoke the custom action "Ec2 PostOpsItem" then open a new browser tab to view the item in Systems Manager's "OpsCenter"

## Module 3 - Automated Remediations - GuardDuty DNS Event on EC2 Instance
1.  Run the following command which runs a policy named [ec2-sechub-remediate-severity-with-findings](https://github.com/FireballDWF/securityhub-remediations/blob/master/module3/ec2-sechub-remediate-severity-with-findings.yml) which instructs Cloud Custodian to dynamically generate and deploy a lambda, which will be invoked when [SecurityHub generates a Cloudwatch Event](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cloudwatch-events.html) when sent a finding. In this module, the finding will be triggered when GuardDuty generates a finding, and the severity of the is greater than or equal to 31, and the EC2 instance has any vulnerability previously reported to SecurityHub
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module3/ec2-sechub-remediate-severity-with-findings.yml
```
2.  Next run the following command which leverages [Systems Manager's Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html) to run an "nslookup" command on the ec2 instance tag:Name RemediationTestTarget where it's looking up a dns name which [GuardDuty will detect as Command and Control activity](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_backdoor.html#backdoor7)
```
aws ssm send-command --document-name AWS-RunShellScript --parameters commands=["nslookup guarddutyc2activityb.com"] --targets "Key=tag:Name,Values=RemediationTestTarget" --comment "Force GuardDutyFinding" --cloud-watch-output-config CloudWatchLogGroupName=/aws/ssm/AWS-RunShellScript,CloudWatchOutputEnabled=true
```
3.  As it can take a long time (more than 20 minutes often around 2 hours) for GuardDuty to generate a DNS based finding, please proceed to the next module, then come back to the next review step later.
4.  Review the Logs via https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/lambda/custodian-ec2-sechub-remediate-severity-with-findings;streamFilter=typeLogStreamPrefix {TODO} fix us-east-1 reference, describe what to do.

## Module 4 - Automated Remediations - Vulnerability Event on EC2 Instance with Very Risky Configuration
1.  Run the following command, which invokes Cloud Custodian to run a policy named [ec2-public-ingress-hubfinding](https://github.com/FireballDWF/securityhub-remediations/blob/master/module4/ec2-public-ingress-hubfinding.yml) which filters for a high risk configuation (Details TODO).
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module4/ec2-public-ingress-hubfinding.yml
```
2.  Run the following command to trigger an finding event in Security Hub on the with the resource being the RemediationTestTarget instance. 
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module1/force-vulnerability-finding.yml  
```
3.  Review the Logs via https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logStream:group=/aws/lambda/custodian-ec2-public-ingress-hubfinding;streamFilter=typeLogStreamPrefix 
4.  Review module4/ec2-public-ingress.yml observing that the lack of a "mode" section means it can be run anytime to find the risky configuration without requiring a vulnerability event.
5.  Now run the following command to re-associate the InstanceProfile so the instance is ready for the next module.
```
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=SecurityHubRemediationWorkshopCli --instance-id $(aws ec2 describe-instances --filters "Name=tag:Name,Values=RemediationTestTarget" --query Reservations[*].Instances[*].[InstanceId] --output text)
```
  
## Module 5 - Automated Remediations - GuardDuty Event on IAMUser
1. Run the following command:
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module5/iam-user-hubfinding-remediate-disable.yml
```
2.  Verify that the previous command resulted in output containing "Provisioning policy lambda iam-user-hubfinding-remediate-disable"
3.  Optional, but at least read: Next archive any existing sample GuardDuty Findings for the IAM User named GeneratedFindingUserName.  While this is not nessesary when the create-sample-findings command (which is run later in this module) has never been run before, it won't harm anything to run.  And if it's not run and the sample finding has already been generated, then the cloudwatch event needed for this module to function never gets triggered, so we're running it to eliminate sources of potential error.  And if you want to rerun this module, you need to run this command.
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
If you get an error like "InternalServerErrorException", the most likely explaination is that there are no findings currently, and if, you can move on the next step.
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
8.  Next, goto the Security Hub console and look the finding Title = "API GeneratedFindingAPIName was invoked from a known malicious IP address" however expect to need to wait for 2-5 minutes for it to appear. 
9.  Next validate the execution of the automated remediation by looking within CloudWatch Logs, remember the LogGroup pattern of /aws/lambda/custodian-$(name of the cloud custodian policy).  You are looking a lines containing "policy:iam-user-hubfinding-remediate-disable invoking action:userremoveaccesskey resources:1" followed by a line containing "metric:ApiCalls Count:2 policy:iam-user-hubfinding-remediate-disable restype:iam-user", if they don't appear will need to troubleshoot using the logs.
10.  Validate that the Access Keys were actually removed from the user by running the following command:
```
aws iam list-access-keys --user-name GeneratedFindingUserName
```
11. Evaluate the output by looking for "Status"="Inactive"


## Module 6 - Remediate an Public EBS-Snapshot 

This module will show how to setup an automated detection of a EBS Snaspshot that has been made public, with a finding submitted to Security Hub, then use Security Hub Custom action to delete the snapshot.  Then we'll fully automate the remediation by changing the detection policy to perform the delete while still providing notification.  
1. Start by reviewing the file "post-ebs-snapshot-public.yml" using the Cloud9 IDE. Observe that the only action type is "post-finding". Also Observe that the following two lines instruct Cloud Custodian to deploy a CloudWatch rule which triggers on a regular schedule, configured to be every 5 minutes.
```
      type: periodic
      schedule: "rate(5 minutes)"
```
2. Then deploy the automated detection policy by running the following command:
```
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module6/post-ebs-snapshot-public.yml
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
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module6/post-ebs-snapshot-public.yml
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
docker run -it --rm --group-add 501 -v /home/ec2-user/environment/securityhub-remediations/output:/home/custodian/output:rw -v /home/ec2-user/environment/securityhub-remediations:/home/custodian/securityhub-remediations:ro -v /home/ec2-user/.aws:/home/custodian/.aws:ro ${SECHUBWORKSHOP_CONTAINER} run --cache-period 0 -s /home/custodian/output -c /home/custodian/securityhub-remediations/module6/post-ebs-snapshot-public.yml
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
docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.actions.post-item
```
TODO: Learn more about post-findings, put in prior section