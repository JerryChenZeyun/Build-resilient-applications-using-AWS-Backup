# **Build resilient applications using AWS Backup and AWS Resilience Hub**

This lab is provided as part of **[AWS Innovate For Every Application Edition](https://aws.amazon.com/events/aws-innovate/apj/for-every-app/)**

Click [here](https://github.com/phonghuule/aws-innovate-fea-2022) to explore the full list of hands-on labs.

ℹ️ You will run this lab in your own AWS account. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Introduction

This lab was derived from one of **Well-Architected Reliability Labs** named [TESTING BACKUP AND RESTORE OF DATA](https://wellarchitectedlabs.com/reliability/200_labs/200_testing_backup_and_restore_of_data/) in AWS Well-Architected Lab with latest update.

The purpose of this lab is to teach you how to use [AWS backup](https://aws.amazon.com/backup/) to achieve automatic data backup, alongside resiliency assessment genearted from [AWS Resilience Hub](https://aws.amazon.com/resilience-hub/) so as to build resilient applications with well defined data backup strategy.

Create backup of the data is just first step to enhance data resilience. You as data owner must also test these backups to ensure they can be used to recover data. A backup is useless if you are unable to restore your data from it. Testing the restore process after each backup will ensure you are aware of any issues that might arise during a restore down the line.

In this lab, you will create an EC2 Instance as a data source. You will then create a strategy to backup these data sources periodically using AWS Backup, and finally, automate the testing of the restore process as well as cleanup of resources using AWS Lambda.

Then you will use AWS Resilience Hub to create application resilience report against the RTO/RPO requirements you define, and receive recommendations from the service to enhance the resiliency of the application.

The skills you learn will help you define a backup and restore plan in alignment with the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/?wa-lens-whitepapers.sort-by=item.additionalFields.sortDate&wa-lens-whitepapers.sort-order=desc&wa-guidance-whitepapers.sort-by=item.additionalFields.sortDate&wa-guidance-whitepapers.sort-order=desc).

## Goals:
* Set up a Backup Strategy to ensure mission-critical data is being backed up regularly
* Test restoring from EC2 image backups to ensure there are no data recovery issues
* Learn how to automate this process
* Getting resiliency assessment report for the application

## Prerequisites:
* An [AWS Account](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html) that you are able to use for testing, that is not used for production or other purposes.
* An IAM user or role in your AWS account that has Administrator privileges. Launch the CloudFormation Stack to provision resources that will act as data sources.

## Note
NOTE: You will be billed for any applicable AWS resources used if you complete this lab that are not covered in the [AWS Free Tier](https://aws.amazon.com/free/).

## Deploy the Infrastructure and Application
You may refer to the following solution architectual view. You will use AWS CloudFormation to provision resources needed for this lab. As part of this lab, the CloudFormation stack that you provision will create an EC2 Instance, an SNS Topic, and a Lambda Function. You can view the CloudFormation template [here](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/aws-backup-lab-res.yaml) for a complete list of all resources that are provisioned. This lab will **only work in us-east-1**.

![Solution Architect View](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/solution-topo1.png)
<br /><br />

### Step1: Deploy the Infrastructure
#### 1.1 Log into the AWS console
Sign in to the AWS Management Console as an IAM user who has PowerUserAccess or AdministratorAccess permissions, to ensure successful execution of this lab.

#### 1.2 Deploy the infrastructure using AWS CloudFormation
Click the following button to deploy the stack. [\
![](https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2019/10/30/LaunchCFN.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=WA-Backup-Lab&templateURL=https://aws-innovate-2022-aws-backup-lab.s3.ap-southeast-2.amazonaws.com/aws-backup-lab-res.yaml)

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
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create_backup_plan_2023.png)

Alternatively, you can click the **Backup plans** in the navigation plane, then click the **Create backup plan** button.

#### 2.3 Select the option to BUILD A NEW PLAN
Specify a **Backup plan name** such as **BACKUP-LAB**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/build-a-new-plan.png)

#### 2.4 Configure the Backup Rule
Give the Backup Rule a name, such as **BACKUP-LAB-RULE**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/BACKUP-RULE-CREATION.png)

##### 2.4.1 Configure Backup vault
Click the **Create new Backup vault** button. 
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/click-create-backup-vault.png)

In the **Create backup vault** window, input the **BACKUP VAULT NAME**, such as **BACKUP-LAB-VAULT**.


You can choose to encrypt your backups for additional security by specifying a KMS key. You can choose the default key created and managed by AWS Backup or specify your own custom key. For this exercise, select the default key (default) aws/backup.


Then click **CREATE BACKUP VAULT**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create-backup-vault-2023.png)

##### 2.4.2 Select the newly created Backup vault
When the Backup Vault has been created, you can switch back to the **Create backup plan** page, and select the newly created Backup Vault in the drop-down list. If you cannot find the new Backup Vault, you can click the **refresh** button.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/select-backup-vault-2023.png)

#### 2.5 Complete the Backup plan creation
Once the Backup Vault has been created, you can set a **SCHEDULE** for the backup, you can specify the **FREQUENCY** at which backups are taken. You can enter frequency as every 12 hours, Daily, Weekly, or Monthly. Alternatively, you can specify a custom **CRON EXPRESSION** for your backup frequency. For this exercise, select the **FREQUENCY** as **DAILY**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/daily-backup-window-2023.png)

Leave the other configuration as **DEFAULT**. And then click **Create Plan** button to complete the backup plan creation.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/complete-backup-plan-creation-2023.png)

Once the backup plan and the backup rule has been created, you can specify resources to back up. You can select individual resources to be backed up, or specify a tag (key-value) associated with the resource. AWS Backup will execute backup jobs on all resources that match the tags specified.

#### 2.6 Set up the Resource Assignment for Backup Plan
You can continue to specify a **RESOURCE ASSIGNMENT NAME** such as **BACKUP-RESOURCES** to help identify the resources that are being backed up.
(If you are not on the **Assign resources** page, then you can click on **BACKUP PLANS** from the menu on the left side of the screen, select the backup plan **BACKUP-LAB** that you just created. Then you will find the **Resource assignments** section, then you can click the **Assign resources** button.)

Leave the **DEFAULT ROLE** selected for **IAM ROLE**. If a role does not already exist, the AWS Backup service will create one with the necessary permissions.

Under **RESOURCE SELECTION**, you can specify resources to be backed up individually by specifying the **RESOURCE TYPE** and **RESOURCE ID**, or select TAGS and enter the TAG KEY and the TAG VALUE. For this lab, select TAGS key as **workload**, Condition for value as **Equals**, and the Value as **myapp**. This tag and value was created by the CloudFormation stack. Remember that tags are case sensitive and ensure that the values you enter are all in lower case.

![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/assign-resources-2023.png)

#### 2.7 Complete Resource Assignment
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

5. Under **RETENTION PERIOD**, select the option DAYS AFTER CREATION and enter 1 for the value for this lab. This will ensure that the backup is deleted after 1 day.

6. Under **Backup Vault**, select the **BACKUP-LAB-VAULT**.

7. Leave the default IAM role selected.

8. Click **CREATE ON-DEMAND BACKUP**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/backup-retention-period-2023.png)

9. Click on **JOBS** from the menu on the left and select **BACKUP JOBS**. You should see a new backup job started with the status of **RUNNING**. Click on the **RESTORE JOBS** tab, there shouldn’t be any restore jobs running.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/backup-job-2023.png)

10. Periodically refresh the console until the **STATUS** changes to **COMPLETED**. It should take about 5-10 minutes to complete.

11. After the job is completed, click on the **JOB ID** and view the **DETAILS**. You should see the **Recovery Point ARN** that was created, the **RESOURCE ID** for which the backup was created, and the **RESOURCE TYPE** for which the backup was created.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/backup-job-completed-2023.png)

12. Monitor your email to see if you receive a **Notification from AWS Backup**. Compare details in the email to what you see on the AWS Console, they should match. It takes about 10 mins for the email to show up once the backup job has completed.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/backup-job-completed-email-2023.png)

**Note: If you haven't received the backup job completion notification email, please check if you have clicked the "Confirm subscription" link in the subscription email sent upon the completion of the CloudFormation as shown below:**
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/confirmed-notification-email-2023.png)

13. The SNS Topic that was created as part of the CloudFormation stack has been configured as a trigger for the Lambda function. When AWS Backup publishes a new message when a backup or restore job is completed, the Lambda function gets invoked.

14. Let’s take a look at the relevant section of the Lambda function code to understand what will happen once the backup job is completed and a notification has been received. You can view the full Lambda function code [here](https://wellarchitectedlabs.com/Reliability/200_Testing_Backup_and_Restore_of_Data/Code/lambda_function.py).

The Lambda function first obtains the [recovery point restore metadata](https://docs.aws.amazon.com/cli/latest/reference/backup/get-recovery-point-restore-metadata.html) for the recovery point that was created when the on-demand backup job was initiated.

```
metadata = backup.get_recovery_point_restore_metadata(
          BackupVaultName=backup_vault_name,
          RecoveryPointArn=recovery_point_arn
      )
```

Once the recovery point restore metadata has been retrieved, the function will then use this to make an API call to AWS Backup to start a restore job.

```
restore_request = backup.start_restore_job(
              RecoveryPointArn=recovery_point_arn,
              IamRoleArn=iam_role_arn,
              Metadata=metadata['RestoreMetadata']
      )
```

15. Go back to **JOBS** and switch to the **RESTORE JOBS** tab. You should see a **RESTORE JOB** running (or completed, depending on when you look at it). The lambda function that was created as part of this lab has requested a restore job from AWS Backup. This is to ensure data recovery from the backup is successful.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/restore-job-completed-2023.png)

16. Note down the RESOURCE ID of the newly created EC2 Instance and verify that it exists from the EC2 Console - <br />https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:sort=instanceState. <br />Note down the public IP of the new EC2 Instance.

17. Monitor your email to see if you have received a **Notification from AWS Backup** confirming the restore job was successful. Compare details in the email to what you see on the AWS Console, they should match. It takes about 10 mins for the email to show up once the restore job has completed.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/backup-job-restore-completed-2023.png)

18. While waiting for the notification, let’s take a look at the relevant sections of the Lambda function code to see what happens when a restore job is completed. You can view the full Lambda function code [here](https://wellarchitectedlabs.com/Reliability/200_Testing_Backup_and_Restore_of_Data/Code/lambda_function.py).

After receiving confirmation from AWS Backup that the restore job has completed successfully, the Lambda function will verify data recovery. To do this, an API call is made to retrieve the public IP address of the new EC2 Instance. It makes an HTTP GET request to the new EC2 Instance to check if the application is running. If a valid response (200 in this case) is received, it is ascertained that data recovery was successful as per the recovery success criteria that was established earlier in this section. The Lambda function will then make an API call to EC2 to terminate the new EC2 Instance to save on cost. You can manually verify this as well by visiting the following URL:

```
http://<PUBLIC_IP_OF_THE_NEW_INSTANCE>/
```
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/lambda-code.png)

19. Monitor your email to see if you have received a **Restore Test Status** notification confirming the deletion of the newly created resource. Check the EC2 Console to verify that the new EC2 Instance has been terminated. - <br />https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Volumes:sort=size

If you see the EC2 instance still up and running, it means the lambda function is in progress to terminate the instance.
Please give it some time and check back again.

#### Review of Best Practices Implemented

**Identify all data that needs to be backed up and perform backups or reproduce the data from sources:** Back up important data using Amazon S3, Amazon EBS snapshots, or third-party software. Alternatively, if the data can be reproduced from sources to meet RPO, you may not require a backup.

**Perform data backup automatically or reproduce the data from sources automatically:** Automate backups or the reproduction from sources using AWS features (for example, snapshots of Amazon RDS and Amazon EBS, versions on Amazon S3, etc.), AWS Marketplace solutions, or third-party solutions.

**Perform periodic recovery of the data to verify backup integrity and processes:** Validate that your backup process implementation meets Recovery Time Objective and Recovery Point Objective through a recovery test.


### Step5: Validating Application Resiliency via AWS Resilience Hub

[AWS Resilience Hub](https://aws.amazon.com/resilience-hub/) is a new service that helps you understand and improve the resiliency of your workloads using AWS Well-Architected best practices.

You can take this additional step to use Resilience Hub to assess and improve the resiliency of the application architecture based on its recommendations.

1. Sign in to the AWS Management Console and navigate to the AWS Resilience Hub console - <br />
(https://us-east-1.console.aws.amazon.com/resiliencehub/home?region=us-east-1#/homepage)


2. Click the **Add Application** button to add the web application you've deployed to AWS Resilience Hub.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resilience-hub-add-app.png)

3. Give it a name to the **Application name**, such as **WEB-WA-APP**, with an app description (optional). Leave all as default in the **How is this application managed** section.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resilience-hub-app-name-2023.png)

4. In the **Add resource collections** section, select **CloudFormation stacks**.

5. Select **WA-Backup-Lab** CloudFormation stack in the **Select stacks**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/select-cfn-stack-2023.png)

6. In the **Set RTO and RPO** section, change the **RPO** to **1 days** in the **RTO/RPO targets** sub section. 
Likewise, change **RPO** to **1 days** in **Infrastructure** and **Availability Zone** sub sections.
In this case, we assume this is a medium business priority application, which can bear with 1 day RPO target.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/rto-rpo-1-day-2023.png)

7. In the **Set up permissions**, choose **Use an IAM role**, and select **AWS-WA-Lab-Resilience-Hub-Role** option in the drop-down list.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resilience-hub-iam-role-2023.png)

8. Leave all settings in the rest sections as default, and click the **Add application** button at the bottom of the page. This will generate an application in the Resilience Hub.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resilience-hub-add-app-2023.png)

9. In the **Applications** page, click the **Publish application** button complete the application publishing.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/application-publishing-2023.png)

10. Once the application is published, click the **Assess resiliency** button to complete the assessment.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resiliency-assessment-2023.png)

11. Click the **Run** button in the pop up window to run the assessment.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/run-assessment-pop-up-2023.png)

12. After couple seconds, the assessment will be completed. You should notice there's a policy breached in the Compliance status. You can click the **Assessment Name** to investigate the Assessment report details.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/policy-breached-2023.png)

13. By investigating the Assessment reports, you can see the reason for policy breach is that the application is unrecoverable. 
And if you click the **Resiliency recommendations** tab, you will find Resilience Hub recommend to set up a dead-letter queue (DLQ) for the Amazon SNS, so to enable subscriptions to re-try the failed deliveries later. In this case, the application can achieve the RPO resilient goal.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/application-unrecoverable-2023.png)

14. To meet the RPO goals, we will configure the dead-letter queue for the SNS subscriptions. Search **SQS** in the console search bar, and go to the Amazon SQS console. Click the **Create queue** button.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/sqs-create-queue-2023.png)

15. Give the Amazon SQS queue a name, such as **WA-dead-letter-queue**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/name-sqs-queue-2023.png)

16. Leave all other settings as default, and click the **Create queue** button to create the Amazon SQS queue.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create-sqs-queue-2023.png)

17. Go back to Amazon SNS console, click the **Subscription** in the menu, search the topic **BackupNotificationTopic-WA-Backup-Lab**, where you will see 2 subsriptions for this topic. Then select the first subscription, and click the **Edit** button.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/edit-sub1-2023.png)

18. In the **Edit subscription** page, extend the **Redrive policy** option, enable the **Redrive policy (dead-letter queue)**, and select the **WA-dead-letter-queue** SQS ARN in the drop-down list. Then click the **Save changes** button.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/config-dlq-sub1-2023.png)

19. Repeat steps 17-18 to enable **Redrive policy (dead-letter queue)** for the other subscription of the SNS topic **BackupNotificationTopic-WA-Backup-Lab**.

20. Once the dead-letter queue has been enabled for both subscriptions of the SNS topic, you can move back to the AWS Resilience Hub console, and click the **Applications** in the menu, select the **WEB-WA-APP** application, and click the **Reassess** button to complete the re-assessment. 
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/reassess-app-2023.png)

21. Click the **Run** button in the pop-up window to execute the re-assessment. After couple seconds, you will see the assessment report, with **Policy met** in the **Compliance status**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/compliant-state-2023.png)

You may exlore the other resiliency and operational recommendations from AWS Resilience Hub. 
Please note that for the sake of simplicity, this lab application utilises a monolithic architecture instead of distributed one. So in practice, you can expect those distributed web application will receive more comprehensive resiliency recommendations from the AWS Resilience Hub.

#### Review of Best Practices Implemented

** You have utilised **AWS Resilience Hub** to automatically collect application information through the Cloudformation stack used to implement the application.
** You have defined a policy to specify RTO/RPO requirements for the specific application
** **AWS Resilience Hub** conducts automatic assessment to generate resiliency report, with the assessment data to showcase if the defined RTO/RPO requirements have been met, alongside relevant optimization suggestions to achieve better application resiliency. 


### Step6: Tear Down

The following instructions will remove the resources that you have created in this lab.

#### Cleaning up AWS Resilience Hub resources

1. Sign in to the AWS Management Console and navigate to the AWS Backup console - <br />https://us-east-1.console.aws.amazon.com/resiliencehub/home?region=us-east-1#/dashboard

2. Click on **Applications** and select the **WEB-WA-APP**, then click **Delete** under the **Actions** button.

3. Type in **Delete** in the pop up window to proceed.

4. Click on **Policies**, select **WA-LAB-RESILIENCY-POLICY**, then click **Delete** under the **Actions** button.

5. Type in **Delete** in the pop up window to proceed.

#### Cleaning up AWS SQS resources

1. Sign in to the AWS Management Console and navigate to the AWS Backup console - <br />https://us-east-1.console.aws.amazon.com/sqs/v2/home?region=us-east-1#/homepage

2. Click on **Queues** and select the **WA-dead-letter-queue** you've created, then click **Delete**, type in **confirm** in the pop-up window, and click the **Delete** button.

#### Cleaning up AWS Backup resources

1. Sign in to the AWS Management Console and navigate to the AWS Backup console - <br />https://us-east-1.console.aws.amazon.com/backup/home?region=us-east-1#home

2. Click on **BACKUP VAULTS** from the menu on the left side, and select **BACKUP-LAB-VAULT**.

3. Under the section **Recovery points**, delete all the RECOVERY POINTS.

4. Once all the **RECOVERY POINTS** have been deleted, delete the **Backup Vault** by clicking on **DELETE VAULT** on the top right hand corner.

5. Click on **BACKUP PLANS** from the menu on the left side, and select **BACKUP-LAB**.

6. Scroll down to the section **RESOURCE ASSIGNMENTS**, and delete the resource assignment.

7. Delete the **BACKUP PLAN** by clicking on **DELETE** on the upper right corner of the screen.

#### Cleaning up the CloudFormation stack

1. Sign in to the AWS Management Console and navigate to the AWS CloudFormation console -<br /> https://console.aws.amazon.com/cloudformation/

2. Select the stack **WA-Backup-Lab**, and delete the stack.

#### Cleaning up the CloudWatch Logs

1. Sign in to the AWS Management Console, and open the CloudWatch console at https://console.aws.amazon.com/cloudwatch/ .

2. Click **Logs** in the left navigation, and then select **Log groups**.

3. Click the check box on the left of the **/aws/lambda/RestoreTestFunction**.

4. Click the **Actions Button** then click **Delete Log Group**.

5. Verify the log group name then click **Delete**.

