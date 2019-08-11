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

1. Environment Build and Configuration
2. 


# Module 1 - Environment Build and Configuration
1. Enable Security Hub (if not already enabled - https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-settingup.html#securityhub-enable
2. Enable Cloud9
3. Create an IAM role for Cloud Custodian
3. Install Cloud Custodian
4. Test Cloud Custodian

# Module 2 - Security Hub Custom Actions - Human initiated automation

# Module 3 - Automated Remediations - GuardDuty Event on EC2 Instance

# Module 4 - Automated Remediations - GuardDuty Event on IAMUser

# Module 5 - Automated Remediations - GuardDuty Event on EC2 Instance - Isolate rather than Stop



