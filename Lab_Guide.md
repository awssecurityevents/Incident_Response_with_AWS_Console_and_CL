# Incident Response with AWS Console and CLI

## Table of Contents

1. [Getting Started](#getting_Started)
2. [Identity & Access Management](#iam)
3. [Amazon VPC](#vpc)
4. [Knowledge Check](#knowledge_check)

## 1. Getting Started <a name="getting_Started"></a>

### 1.1 Start Cloud9

We will use Cloud9 to run all the AWS CLI Commands in this Lab

From the Management Console search for Cloud9
![Cloud9-search](Cloud9-1.png)

This will redirect you to the an environment pre-built called  **incidentreponse**. 
Click on **Open IDE**

![Cloud9-screen](Cloud9-2.1.png)

This will open the web IDE where you can perform any cli based commands from here

![Cloud9-ide](Cloud9-3.png)


### 1.2 Amazon CloudWatch Logs (Run Queries in AWS Cloud9)

[Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) can be used to monitor, store, and access your log files from Amazon Elastic Compute Cloud (Amazon EC2) instances, AWS CloudTrail, Amazon Route 53, Amazon VPC Flow Logs, and other sources. 

It is a [best practice](https://wa.aws.amazon.com/wat.question.SEC_4.en.html) to enable logging and analyze centrally, and develop investigation proceses. Using the AWS CLI and developing runbooks for investigation into different events can be significantly faster than using the console. 

If your logs are stored in Amazon S3 instead, you can use [Amazon Athena](https://docs.aws.amazon.com/athena/latest/ug/what-is.html) to directly analyze data.

To list the Amazon CloudWatch Logs Groups you have configured in each region, you can describe them. Note you must specify the region, if you need to query multiple regions you must run the command for each. 

You must use the region ID such as *ap-southeast-1* instead of the region name of *Asia Pacific (Singapore)* that you see in the console. You can obtain a list of the regions by viewing them in the [AWS Regions and Endpoints](https://docs.aws.amazon.com/general/latest/gr/rande.html) or using the AWS CLI command: 

    aws ec2 describe-regions
    
To list the log groups you have in a region, replace the example ap-southeast-1 with your region:


    aws logs describe-log-groups --region ap-southeast-1


The default output is json, and it will give you all details. If you want to list only the names in a table:

    aws logs describe-log-groups --output table --query 'logGroups[*].logGroupName' --region ap-southeast-1
  
***

## 2. Identity & Access Management <a name="iam"></a>

### 2.1 Investigate AWS CloudTrail

As AWS CloudTrail logs API activity for [supported services](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-aws-service-specific-topics.html), it provides an audit trail of your AWS account that you can use to track history of an adversary. 

For example, listing recent access denied attempts in AWS CloudTrail may indicate attempts to escalate privilege unsuccessfully. Note that some services such as Amazon S3 have their own logging, for example read more about [Amazon S3 server access logging](https://docs.aws.amazon.com/AmazonS3/latest/dev/ServerLogs.html). 

You can enable AWS CloudTrail by following the [Automated Deployment of Detective Controls](https://github.com/awssecurityevents/Automated_Deployment_of_Detective_Controls) lab.

#### 2.1.1 AWS Console (Run Queries in Amazon CloudWatch)

The AWS console provides a visual way of querying Amazon CloudWatch Logs, using [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html) and does not require any tools to be installed.

* Open the Amazon CloudWatch console at [CloudWatch](https://console.aws.amazon.com/cloudwatch/) and select your region.
* From the left menu, choose **Insights** under **Logs**.
* From the dropdown near the top select your CloudTrail Logs group, then the relative time to search back on the right.
* Copy the following example queries below into the query input, then click **Run query**.

**IAM Access Denied Attempts:**
To list all IAM access denied attempts you can use the following example. Each of the line item results allows you to drill down to reveal further details.


    filter errorCode like /Unauthorized|Denied|Forbidden/ | fields awsRegion, userIdentity.arn, eventSource, eventName, sourceIPAddress, userAgent


**IAM access key:**
If you need to search for what actions an access key has performed you can search for it e.g. `AKIAIOSFODNN7EXAMPLE`:

    filter userIdentity.accessKeyId ="AKIAIOSFODNN7EXAMPLE" | fields awsRegion, eventSource, eventName, sourceIPAddress, userAgent
    
**IAM source ip address:**
If you suspect a particular IP address as an adversary you can search such as `192.0.2.1`:


    filter sourceIPAddress = "192.0.2.1" | fields awsRegion, userIdentity.arn, eventSource, eventName, sourceIPAddress, userAgent

 
**IAM access key created**
An access key id will be part of the responseElements when its created so you can query that:

    filter responseElements.credentials.accessKeyId ="AKIAIOSFODNN7EXAMPLE" | fields awsRegion, eventSource, eventName, sourceIPAddress, userAgent
    
**IAM users and roles created**
Listing users and roles created can help identify unauthorized activity:

    filter eventName="CreateUser" or eventName = "CreateRole" | fields requestParameters.userName, requestParameters.roleName, responseElements.user.arn, responseElements.role.arn, sourceIPAddress, eventTime, errorCode
    
**S3 List Buckets**

Listing buckets may indicate someone trying to gain access to your buckets. Note that [Amazon S3 server access logging](https://docs.aws.amazon.com/AmazonS3/latest/dev/ServerLogs.html) needs to be enabled on each bucket to gain further S3 access details:

    filter eventName ="ListBuckets"| fields awsRegion, eventSource, eventName, sourceIPAddress, userAgent
    

#### 2.1.2 AWS CLI (Run Queries in AWS Cloud9)

Remember you might need to update the *--log-group-name*, *--region* and/or *--start-time* parameter to a millisecond epoch start time of how far back you wish to search. You can use a web conversion tool such as [www.epochconverter.com](https://www.epochconverter.com/).


**Install jq in Cloud9**

In Cloud9, please run the following command

    sudo yum install jq -y

**IAM access denied attempts:**

To list all IAM access denied attempts you can use CloudWatch Logs with *--filter-pattern* parameter of `AccessDenied` for roles and `Client.UnauthorizedOperation` for users. 
Replace the --log-group-name with the log group name from your AWS Cloudwatch log group E.g. 'Detective-Controls-CloudTrailCWLogsGroup-XXXXXXXXXXXX'

    aws logs filter-log-events --region ap-southeast-1 --start-time 1551402000000 --log-group-name <your_log_group_name> --filter-pattern AccessDenied --output json --query 'events[*].message'| jq -r '.[] | fromjson | .userIdentity, .sourceIPAddress, .responseElements'
    
**IAM access key:**

If you need to search for what actions an access key has performed you can modify the *--filter-pattern* parameter to be the access key to search such as `AKIAIOSFODNN7EXAMPLE`. Replace the --log-group-name with the log group name from your AWS Cloudwatch log group E.g. 'Detective-Controls-CloudTrailCWLogsGroup-XXXXXXXXXXXX'

    aws logs filter-log-events --region ap-southeast-1 --start-time 1551402000000 --log-group-name <your_log_group_name>  --filter-pattern AKIAIOSFODNN7EXAMPLE --output json --query 'events[*].message'| jq -r '.[] | fromjson | .userIdentity, .sourceIPAddress, .responseElements'

**IAM source ip address:**

If you suspect a particular IP address as an adversary you can modify the *--filter-pattern* parameter to be the IP address to search such as `192.0.2.1`. Replace the --log-group-name with the log group name from your AWS Cloudwatch log group E.g. 'Detective-Controls-CloudTrailCWLogsGroup-XXXXXXXXXXXX'

    aws logs filter-log-events --region ap-southeast-1 --start-time 1551402000000 --log-group-name <your_log_group_name>  --filter-pattern 192.0.2.1 --output json --query 'events[*].message'| jq -r '.[] | fromjson | .userIdentity, .sourceIPAddress, .responseElements'
    
**S3 List Buckets**

Listing buckets may indicate someone trying to gain access to your buckets. Note that [Amazon S3 server access logging](https://docs.aws.amazon.com/AmazonS3/latest/dev/ServerLogs.html) needs to be enabled on each bucket to gain further S3 access details. Replace the --log-group-name with the log group name from your AWS Cloudwatch log group E.g. 'Detective-Controls-CloudTrailCWLogsGroup-XXXXXXXXXXXX'

    aws logs filter-log-events --region ap-southeast-1 --start-time 1551402000000 --log-group-name <your_log_group_name>  --filter-pattern ListBuckets --output json --query 'events[*].message'| jq -r '.[] | fromjson | .userIdentity, .sourceIPAddress, .responseElements'
    

### 2.2 Block access in AWS IAM

Blocking access to an IAM entity, that is a role, user or group can help when there is unauthorized activity as it will no longer be able to perform any actions. Be careful as blocking access may disrupt the operation of your workload, which is why it is important to practice in a non-production environment. 

Note that the AWS IAM entity may have created another entity, or other resources that may allow access to your account. You can use [AWS CloudTrail](https://aws.amazon.com/cloudtrail/) that logs activity in your AWS account to determine the IAM entity that is performing the unauthorized operations. Additionally [service last accessed data](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_access-advisor.html) in the AWS Console can help you audit permissions.


### 2.3 List AWS IAM roles/users/groups

If you need to confirm the name of a role, user or group you can list:

#### 2.3.1 AWS Console

* Sign in to the AWS Management Console as an IAM user or role in your AWS account, and open the AWS IAM console at [IAM Console](https://console.aws.amazon.com/iam/).
* Click Roles on the left, the role will be displayed and you can use the search field.

#### 2.3.2 AWS CLI (Run Queries in AWS Cloud9)

    aws iam list-roles
    
This provides a full json formatted list of all roles, if you only want to display the *RoleName* use an output of table and query:

    aws iam list-roles --output table --query 'Roles[*].RoleName'
    
List all users:

    aws iam list-users --output table --query 'Users[*].UserName'
    
List all groups:

    aws iam list-groups --output table --query 'Groups[*].GroupName'
    

### 2.4 Attach inline deny policy

Attaching an explicit deny policy to an AWS IAM role, user or group will quickly block **ALL** access for that entity which is useful if it is performing unauthorized operations.

#### 2.4.1 AWS Console

* Sign in to the AWS Management Console as an AWS IAM user or role in your AWS account, and open the AWS IAM console at [https://console.aws.amazon.com/iam/](https://console.aws.amazon.com/iam/).
* Click on **Policies**  on the left, then click create policy.
* Click the **JSON** tab then replace the example with the following:
    
```
{
    "Version": "2012-10-17",
    "Statement": [
        { "Effect": "Deny", 
        "Action": "*", 
        "Resource": "*" }
        ]
}
```
* Click **Review policy**.
* Enter name of **DenyAll** then click **Create policy**.

#### 2.4.2 AWS CLI (Run Queries in AWS Cloud9)
Before you run the commands, you will need to update the temporary keys with your Event Engine credentials.
Browse to [Event Engine](https://dashboard.eventengine.run) and click on **AWS Console** and copy the credentails section and paste that in Cloud9


Create a new group

    aws iam create-group --group-name DenyGroup

Create a new user

    aws iam create-user --user-name DenyUser

Add user to group

    aws iam add-user-to-group --user-name DenyUser --group-name DenyGroup
    
Create policy document

```
cat >> denypolicy.json <<EOL
{
   "Version": "2012-10-17",
   "Statement": [
   {
     "Effect": "Allow",
     "Principal": {
        "Service": "ec2.amazonaws.com"
     },
   "Action": "sts:AssumeRole"
   }
 ]
}  
EOL
```
    
Create role

    aws iam create-role --role-name denyrole --assume-role-policy-document file://denypolicy.json


Block a role:

    aws iam put-role-policy --role-name denyrole --policy-name DenyAll --policy-document '{ "Statement": [ { "Effect": "Deny", "Action": "*", "Resource": "*" } ] }'
    
Block a user:

    aws iam put-user-policy --user-name DenyUser --policy-name DenyAll --policy-document '{ "Statement": [ { "Effect": "Deny", "Action": "*", "Resource": "*" } ] }'


Block a group:

    aws iam put-group-policy --group-name DenyGroup --policy-name DenyAll --policy-document '{ "Statement": [ { "Effect": "Deny", "Action": "*", "Resource": "*" } ] }'

### 2.5 Delete inline deny policy

To delete the policy you just attached and restore the original permissions the entity had:

#### 2.5.1 AWS Console

* Sign in to the AWS Management Console as an IAM user or role in your AWS account, and open the AWS IAM console at [IAM Console](https://console.aws.amazon.com/iam/).
* Click **Roles** on the left.
* Click the checkbox next to the role to delete.
* Click **Delete role**.
* Confirm the role to delete then click **Yes, delete**

#### 2.5.2 AWS CLI (Run Queries in AWS Cloud9)

Delete policy from a role:

    aws iam delete-role-policy --role-name denyrole --policy-name DenyAll
    
Delete policy from a user:

    aws iam delete-user-policy --user-name DenyUser --policy-name DenyAll
    
Delete policy from a group:

    aws iam delete-group-policy --group-name DenyGroup --policy-name DenyAll

## 3. Amazon VPC <a name="vpc"></a>

A Amazon VPC that has [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html) enabled captures information about the IP traffic going to and from network interfaces in your Amazon VPC. 

This log information may help you investigate how Amazon EC2 instances and other resources in your VPC are communicating, and what they are communicating with. You can follow the [Automated Deployment of VPC](https://github.com/awslabs/aws-well-architected-labs/tree/master/Security/200_Automated_Deployment_of_VPC) lab for creating a Amazon VPC with Flow Logs enabled.

### 3.1 Investigate Amazon VPC Flow Logs

#### 3.1.1 AWS Management Console (Run Queries in Amazon CloudWatch)

The AWS Management console provides a visual way of querying CloudWatch Logs, using [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html) and does not require any tools to be installed.

* Open the Amazon CloudWatch console at [CloudWatch Console](https://console.aws.amazon.com/cloudwatch/) and select your region.
* From the left menu, choose **Insights** under **Logs**.
* From the dropdown near the top select your CloudTrail Logs group, then the relative time to search back on the right.
* Copy the following example queries below into the query input, then click **Run query**.

**Rejected requests by IP address:**

Rejected requests indicate attempts to gain access to your VPC, however there can often be noise from internet scanners. To count the rejected requests by source IP address:

    filter action="REJECT"
    | stats count(*) as numRejections by srcAddr
    | sort numRejections desc

**Reject requests originating from inside your VPC**

Rejected requests that originate from inside your VPC may indicate your infrastructure in your VPC is attempting to connect to something it is not allowed to, e.g. a database instance is trying to connect to the internet and is blocked. This example uses regex to match the start of your VPC as *10.*:

    filter action="REJECT" and srcAddr like /^10\./
    | stats count(*) as numRejections by srcAddr
    | sort numRejections desc

**Requests from an IP address**

If you suspect an IP address and want to list all requests that originate, replace *192.0.2.1* with the IP you suspect:

    filter srcAddr = "192.0.2.1"
    | fields @timestamp, interfaceId, dstAddr, dstPort, action
    
**Request count from a private IP address by destination address**

If you want to list and count all connections by a private IP address, replace *10.1.1.1* with your private IP:

    filter srcAddr = "10.1.1.1"
    | stats count(*) as numConnections by dstAddr
    | sort numConnections desc

## 4. Knowledge Check <a name="knowledge_check"></a>

The security best practices followed in this lab are: <a name="best_practices"></a>

* [Analyze logs centrally](https://wa.aws.amazon.com/wat.question.SEC_4.en.html) Amazon CloudWatch is used to monitor, store, and access your log files. You can use AWS CloudWatch to analyze your logs centrally.
* [Automate alerting on key indicators](https://wa.aws.amazon.com/wat.question.SEC_4.en.html) AWS CloudTrail, AWS Config,Amazon GuardDuty and Amazon VPC Flow Logs provide insights into your environment.
* [Implement new security services and features:](https://wa.aws.amazon.com/wat.question.SEC_5.en.html) New features such as Amazon VPC Flow Logs have been adopted.
* [Implement managed services:](https://wa.aws.amazon.com/wat.question.SEC_7.en.html)
Managed services are utilized to increase your visibility and control of your environment.
* [Identify Tooling](https://wa.aws.amazon.com/wat.question.SEC_11.en.html)
Using the AWS Management Console and/or AWS CLI tools with prepared scripts will assist in your investigations.

***

## References & useful resources

[AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/)
[AWS Identity and Access Management User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)
[CloudWatch Logs Insights Query Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)

***

## License

Licensed under the Apache 2.0 and MITnoAttr License.

Copyright 2019 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Apache License, Version 2.0 (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at

    https://aws.amazon.com/apache2.0/

or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
