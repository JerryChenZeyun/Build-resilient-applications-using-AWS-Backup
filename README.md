# **Build resilient applications using AWS Backup**

This lab is provided as part of **[AWS Innovate For Every Application Edition](https://aws.amazon.com/events/aws-innovate/apj/for-every-app/)**

Click [here](https://github.com/phonghuule/aws-innovate-fea-2022) to explore the full list of hands-on labs.

ℹ️ You will run this lab in your own AWS account. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Introduction
The purpose of this lab is to teach you how to use [AWS backup](https://aws.amazon.com/backup/) to achieve automatic data backup, so as to build resilient applications with well defined data backup strategy.

Create backup of the data is just first step to enhance data resilience. You as data owner must also test these backups to ensure they can be used to recover data. A backup is useless if you are unable to restore your data from it. Testing the restore process after each backup will ensure you are aware of any issues that might arise during a restore down the line.

In this lab, you will create an EC2 Instance as a data source. You will then create a strategy to backup these data sources periodically using AWS Backup, and finally, automate the testing of the restore process as well as cleanup of resources using AWS Lambda.

The skills you learn will help you define a backup and restore plan in alignment with the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/?wa-lens-whitepapers.sort-by=item.additionalFields.sortDate&wa-lens-whitepapers.sort-order=desc&wa-guidance-whitepapers.sort-by=item.additionalFields.sortDate&wa-guidance-whitepapers.sort-order=desc).

## Goals:
* Set up a Backup Strategy to ensure mission-critical data is being backed up regularly
* Test restoring from EC2 image backups to ensure there are no data recovery issues
* Learn how to automate this process

## Prerequisites:
* An [AWS Account](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html) that you are able to use for testing, that is not used for production or other purposes.
* An IAM user or role in your AWS account that has Administrator privileges. Launch the CloudFormation Stack to provision resources that will act as data sources.

## Note
NOTE: You will be billed for any applicable AWS resources used if you complete this lab that are not covered in the [AWS Free Tier](https://aws.amazon.com/free/).

## Deploy the Infrastructure and Application
You may refer to the following solution architectual view. You will use AWS CloudFormation to provision resources needed for this lab. As part of this lab, the CloudFormation stack that you provision will create an EC2 Instance, an SNS Topic, and a Lambda Function. You can view the CloudFormation template [here](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/aws-backup-lab.yaml) for a complete list of all resources that are provisioned. This lab will **only work in us-east-1**.

![Solution Architect View](/images/solution-topo.png)
<br /><br />

### Step1: Deploy the Infrastructure
#### 1.1 Log into the AWS console
Sign in to the AWS Management Console as an IAM user who has PowerUserAccess or AdministratorAccess permissions, to ensure successful execution of this lab.

#### 1.2 Deploy the infrastructure using AWS CloudFormation
Click the following button to deploy the stack. [\
![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2019/10/30/LaunchCFN.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=WA-Backup-Lab&templateURL=https://aws-innovate-2022-aws-backup-lab.s3.ap-southeast-2.amazonaws.com/aws-backup-lab.yaml)

Under **PARAMETERS**:

1. Select an **AvailabilityZone** to launch the resources in.
2. For **LatestAmiId** leave the default value. This will automatically retrieve the latest AMI ID for Amazon Linux 2.
3. Specify an email address that you have access to for **NotificationEmail**.

Check the box **I acknowledge that AWS CloudFormation might create IAM resources**.

Click CREATE / CREATE STACK.

The stack takes about 2 minutes to create all the resources. Periodically refresh the page until you see that the **STACK STATUS** is in **CREATE_COMPLETE**. Once the stack is in **CREATE_COMPLETE**, visit the **OUTPUTS** section for the stack and note down the **KEY** and **VALUE** for each of the outputs. This information will be used later in the lab.

You can view the simple application running on the instance by visiting the URL specified in the outputs. If you get an error that says “Connection refused”, wait a couple of minutes and try again.

### Step2: Create a Backup Plan
Now you will create a backup strategy by leveraging AWS Backup, a fully managed backup service that can automatically backup data from various data sources such as EC2 Instances, EBS Volumes, RDS Databases, and more. You can view a complete list of supported services [here](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html#supported-resources).

#### 2.1 [Sign in to the AWS Backup console](https://us-east-1.console.aws.amazon.com/backup/home?region=us-east-1#backupplan)

#### 2.2 Choose "CREATE BACKUP PLAN"
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create_backup_plan.png)

#### 2.3 Select the option to BUILD A NEW PLAN
Specify a **Backup plan name** such as **BACKUP-LAB**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/build-a-new-plan.png)

#### 2.4 Configure the Backup Rule
Give the Backup Rule a name, such as **BACKUP-LAB-RULE**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/BACKUP-RULE-CREATION.png)

##### 2.4.1 Configure Backup vault
Click the **Create new Backup vault** button. 
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/click-create-backup-vault.png)

In the popup window, input the **BACKUP VAULT NAME**, such as **BACKUP-LAB-VAULT**.


You can choose to encrypt your backups for additional security by specifying a KMS key. You can choose the default key created and managed by AWS Backup or specify your own custom key. For this exercise, select the default key (default) aws/backup.


Then click **CREATE BACKUP VAULT**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create-backup-vault-popup.png)

#### 2.5 Complete the Backup plan creation
Once the Backup Vault has been created, you can set a **SCHEDULE** for the backup, you can specify the **FREQUENCY** at which backups are taken. You can enter frequency as every 12 hours, Daily, Weekly, or Monthly. Alternatively, you can specify a custom **CRON EXPRESSION** for your backup frequency. For this exercise, select the **FREQUENCY** as **DAILY**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/rest-part-of-backup-plan-creation.png)

Leave the other configuration as **DEFAULT**. And then click **Create Plan** button to complete the backup plan creation.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create-backup-rule-plan.png)

Once the backup plan and the backup rule has been created, you can specify resources to back up. You can select individual resources to be backed up, or specify a tag (key-value) associated with the resource. AWS Backup will execute backup jobs on all resources that match the tags specified.

#### 2.6 Review the BACKUP PLAN that's been created
Click on **BACKUP PLANS** from the menu on the left side of the screen. Select the backup plan **BACKUP-LAB** that you just created.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/backup-lab-assign-resource.png)

#### 2.7 Set up the Resource Assignment for Backup Plan
Specify a **RESOURCE ASSIGNMENT NAME** such as **BACKUP-RESOURCES** to help identify the resources that are being backed up.

Leave the **DEFAULT ROLE** selected for **IAM ROLE**. If a role does not already exist, the AWS Backup service will create one with the necessary permissions.

Under **RESOURCE SELECTION**, you can specify resources to be backed up individually by specifying the **RESOURCE TYPE** and **RESOURCE ID**, or select TAGS and enter the TAG KEY and the TAG VALUE. For this lab, select TAGS key as **workload**, Condition for value as **Equals**, and the Value as **myapp**. This tag and value was created by the CloudFormation stack. Remember that tags are case sensitive and ensure that the values you enter are all in lower case.

![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resource-assignment.png)

#### 2.8 Complete Resource Assignment
Click the **Continue** button to complete the Resource Assignment.

![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resource-assignment-continue.png)

You have successfully created a backup plan for your data sources, and all supported resources with the tags workload=myapp will be backed up automatically, at the frequency specified.

### Step3: Enable Notification

In the cloud, setting up notifications to be aware of events within your workload is easily achieved. AWS Backup leverages AWS SNS to send notifications related to backup activities that are occurring. This will allow visibility into backup job statuses, restore job statuses, or any failures that may have occurred, allowing your Operations teams to respond appropriately.

1. Open a terminal where you have access to the AWS CLI. Ensure that the CLI is up to date and that you have AWS Administrator Permissions to run AWS CLI commands.
https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html

2. Edit the following AWS CLI command and include the **ARN** of the **SNS TOPIC** that you created. Replace with the **ARN** of the **SNS TOPIC** obtained from the outputs section of the CloudFormation Stack. **Note that the backup vault name is case sensitive.**
```
aws backup put-backup-vault-notifications --region us-east-1 --backup-vault-name BACKUP-LAB-VAULT --backup-vault-events BACKUP_JOB_COMPLETED RESTORE_JOB_COMPLETED --sns-topic-arn <YOUR SNS TOPIC ARN>
```

3. Once edited, run the above command, it will enable notifications with messages published to the **SNS TOPIC** every time a backup or restore job is completed. This will ensure the Operations team is aware of any failures with backing up or restoring data.

4. You can verify that notifications have been enabled by running the following command. The output will include a section called **SNSTopicArn** followed by the ARN of the SNS Topic that was created as part of the lab.

```
aws backup get-backup-vault-notifications --backup-vault-name BACKUP-LAB-VAULT --region us-east-1
```

![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/verify-sns.png)

You have now successfully enabled notifications for the backup vault BACKUP-LAB-VAULT, ensuring that the Operations team is aware of completion of backup and restore activities involving this vault, and any failures associated with those activities.

### Step4: Test Restoration

For the purpose of this lab, we will simulate the action performed by AWS Backup when creating backups of data sources by creating an on-demand backup to see if the backup is successful. Once the backup is completed, you will receive a notification stating that the backup job has completed and the lambda function will get invoked. The Lambda function will make API calls to start restoring data from the backup that was created. This will help ascertain that the backup is good. Once the restore process has been completed, you will receive another notification confirming this, and the lambda function will get invoked again to clean up new resources that were created as part of the restore. Once the cleanup has been completed, you will receive one last notification confirming cleanup.

1. Use your administrator account to access the AWS Backup console<br />
https://us-east-1.console.aws.amazon.com/backup/home?region=us-east-1#home

2. Click on **CREATE AN ON-DEMAND BACKUP** in the middle of the screen.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/on-demand-backup.png)

3. Under **RESOURCE TYPE**, select **EC2**. Paste in the **Instance ID** obtained from the Output of the CloudFormation Stack.

4. Under **BACKUP WINDOW**, ensure that the **CREATE BACKUP NOW** option is selected.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create-on-demand-backup.png)


### Step5: Tear Down















