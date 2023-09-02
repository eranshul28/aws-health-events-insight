## **AWS Health Events Intelligence Dashboards and Insights (HEIDI)**
Single pane of glass for all your AWS Health events across different accounts,regions and organizations.

## **Table of Contents**
- [Introduction](#introduction)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
    - [Central Account Setup](#central-account-setup)
    - [Member Setup](#member-setup)
- [Update Metadata](#update-metadata)
- [Testing](#testing)
- [Troubleshooting Steps](#troubleshooting-steps)

## **Introduction**

AWS Health Events Intelligence Dashboards and Insights (HEIDI) solution offers a centralized approach to store and analyze AWS Health events. AWS Health serves as the primary means to inform users of service degradation, planned modifications, and issues that may impact their AWS resources. Teams or engineers can go to the AWS health dashboard to find open events, historical events, or scheduled changes. However, they may not always have access to the AWS Health console, and managing communications across multiple accounts, organizations and regions can be difficult. HEIDI provides single pane of glass across different different accounts, regions and organizations. 

The following illustration shows the sample dashboard:

 ![ALT](img/sample.jpg)

## **Solution Architecture**

HEIDI Data Collection Framework enables you to collect data from different different accounts, regions and organizations. AWS Health events are generated by the AWS Health service in each account and region. These events include issues, scheduled maintenances, and account notifications. The AWS Health events are sent to the default Amazon EventBridge bus in the account and region where they are generated. An Amazon Event Bridge rule is set up on the default event bus to forward the events to an central event health bus. This helps to centralize the events and simplify management. Another Amazon EventBridge rule is set up on the central event health bus to send events to Amazon Kinesis Firehose which then put events in S3. Amazon QuickSight is used to consume the consolidated events stored in S3 and display them on a customizable dashboard. 

 ![ALT](img/HeidiDataCollection.jpg)

## **Prerequisites**

1. To backfill the events the solution use AWS Health API. You need to have a Business Support, Enterprise On-Ramp or Enterprise Support plan from AWS Support in order to use this API.
2. Sign up for Amazon QuickSight if you have never used it in this account. To use the forecast capability in QuickSight, sign up for the Enterprise Edition.
3. Verify Amazon QuickSight service has access to Amazon Athena. To enable, go to security and permissions under *manage QuickSight*.
4. AWS Health Event Dashboard will use [SPICE](https://docs.aws.amazon.com/quicksight/latest/user/spice.html) to hold data. Go to SPICE capacity under manage QuickSight and verify you have required space.
5. (Optional) You can enrich incoming AWS Health Events with data such as Resource Tags, Availability Zone, Resource ARN etc. via AWS Config. See [enrichEvent.md](https://github.com/aws-samples/aws-health-events-insight/blob/main/enrichEvent.md) for more details. 

## **Installation**

In this section, We are going to walk through the procedures for configuring Heidi within both the central and member accounts for AWS Health Events.

### **Central Account Setup**

The setup script provided in this repo will set up all the necessary components required to receive AWS health events from other accounts. This can be payer or any other regular AWS account which would receive AWS Health data from all other accounts and regions. 

1. To start, Login to your AWS account and launch `AWS CloudShell` and clone aws-health-events-insight repo.

        git clone https://github.com/aws-samples/aws-health-events-insight.git

2. Go to aws-health-events-insight directory and run OneClickSetup.py and provide account specific inputs.

        cd aws-health-events-insight/src
        python3 OneClickSetup.py

3.  Once CloudFormation status changes to **CREATE_COMPLETE** (about 10-15 minutes), go to Amazon QuickSight dashboard and verify the analysis deployed. 

### **Member Setup**

You MUST complete Member Setup for each Region and Account for which you want to receive AWS Health events.

### Using One click Script(Option1)
1. Setup AWS credntials for desired Account and Regions.
2. Go to aws-health-events-insight directory and run python3 OneClickSetup.py and provide necessary inputs. 

        cd aws-health-events-insight/src
        python3 OneClickSetup.py

### Bulk deployment via StackSet(Option 2)
1. In CloudFormation Console create a stackset with new resources from the template file [HealthEventMember.yaml](https://github.com/aws-samples/aws-health-events-insight/blob/main/src/AWSHealthModule/cfnTemplates/HealthEventMember.yaml).
2. Input the DataCollectionBusArn. Go to the AWS CloudFormation console of central account and get this information from output of DataCollectionStack.
3. Select deployment targets (Deploy to OU or deploy to organization).
4. Select regions to deploy.
5. Submit.

**Note:** To receive global events, you must create Member Account/Region Setup for the US East (N. Virginia) Region and US West (Oregon) Region as backup Region if needed.

## **Update Metadata**
This is an optional stap. You can map AWS AccountIDs with Account Name and Account Tags (AppID, Env, etc.)

1. Go to AWS organizations and export the account list. This is delegated AWS organization account. If you don't have access or somehow can't get the export list, you can create one from [this sample file](https://github.com/aws-samples/aws-health-events-insight/blob/main/src/AWSHealthModule/accountinfo-metadata/Organization_accounts_information_sample.csv).

   ![Export Account List](img/exportAccountList.jpg)

2. For Account Tag, open the file in Excel and add the Tag field as shown below.

   ![Account Tag](img/AccountTag.jpg)

3. **Important:** Upload the file to a specific S3 location so that Amazon QuickSight dataset can join to create mapping.

   ![S3 Location](img/s3Location.jpg)

## **Testing**
Send a mock event to test setup.

1. Go to Amazon EventBridge console and chose default event bus. (You can chose any member account or region) and select send events.
2. **Important** Put the `event source` and `Detail Type` as **"awshealthtest"** , otherwise the rule will discard mock event.
3. Copy the json from [MockEvent.json](https://github.com/aws-samples/aws-health-events-insight/blob/main/src/MockEvent.json) and paste it in the events field, hit send

You will see the event in Amazon S3. For event to reflect in Amazon QuickSight analysis, make sure you refresh the Amazon QuickSight dataset.

## **Troubleshooting Steps**

#### ***1. SYNTAX_ERROR: line 41:15: UNNEST on other than the right side of CROSS JOIN is not supported***

This implies that you are using Amazon Athena V2. Please upgrade worker to Amazon Athena V3. Amazon Athena V2 is on deprecated path.

#### ***2. Template format error: Unrecognized resource types: [AWS::QuickSight::RefreshSchedule]***

`AWS::QuickSight::RefreshSchedule` doesn't exist in certain regions such as us-west-1, ca-central-1 etc. You can comment out `AWSHealthEventQSDataSetRefresh` section in [AWSHealthEventQSDataSet.yaml](https://github.com/aws-samples/aws-health-events-insight/blob/main/src/AWSHealthModule/cfnTemplates/QSDataSetHealthEvent.yaml) and setup refresh schedule from QuickSight console. 

#### ***3. Resource handler returned message: Insufficient permissions to execute the query. Insufficient Lake Formation permission(s) on awshealthevent***

In case Lakeformation is enabled, both the Amazon QuickSight Analysis author and the Amazon QuickSight Service Role need to provide access permissions for the awshealthdb database and all associated tables.

1. Navigate to Lakeformation and go to the "Permissions" tab.
2. Under "Data Lake Permissions," select "Grant."
3.  Choose "SAML users and groups."
4. **Important:** Provide the Amazon QuickSight ARN. This ARN represents the role that owns (or authors) the dataset.
5. From the dropdown menu, select the "awshealthdb" database and grant the necessary permission.
6. Repeat the previous step (Step 5), but this time, select all tables and grant the required permission.
Repeat same process for Amazon QuickSight Service Role.

#### ***4. Possible Reasons for No Data in AWS QuickSight Analysis:***

1. Your AWS environment is relatively new and does not currently have any AWS Health Events. To verify this, please check the AWS Health Dashboard on the AWS Console and send mock event.
2. The Amazon QuickSight DataSet was created before the event could be backfilled by Amazon Kinesis Firehose. To resolve this, manually refresh the Amazon QuickSight DataSet.

[![GitHub Clones](https://img.shields.io/badge/dynamic/json?color=success&label=Clone&query=count&url=https://gist.githubusercontent.com/bajwkanw/24109c8c210fc89367f044d83d07c1bc/raw/clone.json&logo=github)](https://github.com/aws-samples/aws-health-events-insight)