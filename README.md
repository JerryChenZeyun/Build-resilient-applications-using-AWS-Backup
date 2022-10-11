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
You may refer to the following solution architectual view. You will use AWS CloudFormation to provision resources needed for this lab. As part of this lab, the CloudFormation stack that you provision will create an EC2 Instance, an SNS Topic, and a Lambda Function. You can view the CloudFormation template [here](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/aws-backup-lab.yaml) for a complete list of all resources that are provisioned. This lab will **only work in us-east-1**.

![Solution Architect View](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/solution-topo1.png)
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

5. Under **RETENTION PERIOD**, select the option DAYS AFTER CREATION and enter 1 for the value for this lab. This will ensure that the backup is deleted after 1 day.

6. Under **Backup Vault**, select the **BACKUP-LAB-VAULT**.

7. Leave the default IAM role selected.

8. Click **CREATE ON-DEMAND BACKUP**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/complete-on-demand-backup-creation.png)

9. Click on **JOBS** from the menu on the left and select **BACKUP JOBS**. You should see a new backup job started with the status of **RUNNING**. Click on the **RESTORE JOBS** tab, there shouldn’t be any restore jobs running.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/backup-job-dashboard.png)

10. Periodically refresh the console until the **STATUS** changes to **COMPLETED**. It should take about 5-10 minutes to complete.

11. After the job is completed, click on the **JOB ID** and view the **DETAILS**. You should see the **Recovery Point ARN** that was created, the **RESOURCE ID** for which the backup was created, and the **RESOURCE TYPE** for which the backup was created.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/backup-job-completed.png)

12. Monitor your email to see if you receive a **Notification from AWS Backup**. Compare details in the email to what you see on the AWS Console, they should match. It takes about 10 mins for the email to show up once the backup job has completed.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/sns-notification.png)

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
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/restore-job-completed.png)

16. Note down the RESOURCE ID of the newly created EC2 Instance and verify that it exists from the EC2 Console - <br />https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:sort=instanceState. <br />Note down the public IP of the new EC2 Instance.

17. Monitor your email to see if you have received a **Notification from AWS Backup** confirming the restore job was successful. Compare details in the email to what you see on the AWS Console, they should match. It takes about 10 mins for the email to show up once the restore job has completed.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/sns-restore-job-complete.png)

18. While waiting for the notification, let’s take a look at the relevant sections of the Lambda function code to see what happens when a restore job is completed. You can view the full Lambda function code [here](https://wellarchitectedlabs.com/Reliability/200_Testing_Backup_and_Restore_of_Data/Code/lambda_function.py).

After receiving confirmation from AWS Backup that the restore job has completed successfully, the Lambda function will verify data recovery. To do this, an API call is made to retrieve the public IP address of the new EC2 Instance. It makes an HTTP GET request to the new EC2 Instance to check if the application is running. If a valid response (200 in this case) is received, it is ascertained that data recovery was successful as per the recovery success criteria that was established earlier in this section. The Lambda function will then make an API call to EC2 to terminate the new EC2 Instance to save on cost. You can manually verify this as well by visiting the following URL:

```
http://<PUBLIC_IP_OF_THE_NEW_INSTANCE>/
```
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/lambda-code.png)

19. Monitor your email to see if you have received a **Restore Test Status** notification confirming the deletion of the newly created resource. Check the EC2 Console to verify that the new EC2 Instance has been terminated. - <br />https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Volumes:sort=size

20. Use your administrator account to access the AWS CloudWatch console -<br /> https://console.aws.amazon.com/cloudwatch/home?region=us-east-1

21. Click on **LOGS** from the menu on the left side.

22. For filter, paste the following string after replacing the value for the name of the CloudFormation stack that was created as part of this lab.

```
/aws/lambda/RestoreTestFunction-<YOUR CLOUDFORMATION STACK NAME>
```

23. Click on the **LOG STREAM** and view the output of the Lambda function’s execution to understand the different steps performed by the function to automate this process.

#### Review of Best Practices Implemented

**Identify all data that needs to be backed up and perform backups or reproduce the data from sources:** Back up important data using Amazon S3, Amazon EBS snapshots, or third-party software. Alternatively, if the data can be reproduced from sources to meet RPO, you may not require a backup.

**Perform data backup automatically or reproduce the data from sources automatically:** Automate backups or the reproduction from sources using AWS features (for example, snapshots of Amazon RDS and Amazon EBS, versions on Amazon S3, etc.), AWS Marketplace solutions, or third-party solutions.

**Perform periodic recovery of the data to verify backup integrity and processes:** Validate that your backup process implementation meets Recovery Time Objective and Recovery Point Objective through a recovery test.


### Step5: Validating Application Resiliency via AWS Resilience Hub

[AWS Resilience Hub](https://aws.amazon.com/resilience-hub/) is a new service that helps you understand and improve the resiliency of your workloads using AWS Well-Architected best practices.

You can take this additional step to use Resilience Hub to assess and improve the resiliency of the application architecture based on its recommendations.

1. Sign in to the AWS Management Console and navigate to the AWS Backup console - <br />
(https://us-east-1.console.aws.amazon.com/resiliencehub/home?region=us-east-1#/homepage)


2. Click the **Add Application** button to add the web application you've deployed to AWS Resilience Hub.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resilience-hub-add-app.png)

3. In the **Discover application structure** page, select **CloudFormation stacks**.

4. Select **WA-Backup-Lab** CloudFormation stack in the **Select stacks**.

5. Give it a name to the **Application name**, such as **WEB-WA-APP**, with an app description (optional).

6. Tick the check box under the **Scheduled assessment**.

7. Review the above settings, then click **Next** button.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/discover-app-structure.png)

8. In the **Identify resources** page, it might take sometime to discover all existing resources from the Applciation Cloudformation Stack. Once it's done, all relevant resources will be shown up in the page as below. Review the resources (EC2 Instance and Lambda Function) then click **Next**.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/identify-resources.png)

9. In the **Select policy** page, you can click the **Create resiliency policy** button to create the resiliency policy.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create-policy.png)

10. Choose the **Select a policy based on a suggested policy**, so it allows you to use a RTO/RPO template to define the resiliency policy.

11. Give the policy a name, e.g. **WA-LAB-RESILIENCY-POLICY**, with a description (optional).

12. Under the **Suggested resiliency policies**, select the **Important Application**. This sets the RTO (2 days) and RPO (4 hours) for the resiliency policy. Then click **Create** button.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/create-resiliency-policy-details.png)

13. Select the **WA-LAB-RESILIENCY-POLICY**, then click **Next** button.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/select-policy-button.png)

14. Review the application configuration, and then click **Publish** button in the **Review and publish** page.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/publish-app-with-policy.png)

15. Click the **Assess resiliency** button in the application page to initiate the application resiliency assessment through AWS Resilience Hub. Click **Run** button in the pop up window to confirm.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/assess-resiliency.png)

16. After couple seconds, the assessment report will be generated. click the report in the **Resiliency assessment** section.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/assessment-report.png)

17. You will find the assessment result in the report, which shows that the **WEB-WA-APP** application setup meets with the preset RTO/RPO requirements. It will also shows the estimated time for RTO/RPO. There're other metrics for cloud infrastructure RTO/RPO and Availability Zone RTO/RPO data available for reference.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/rto-rpo-result.png)

18. Click on the **Resiliency recommendations** tab, you can find out more recommendations automatically generated by AWS Resilience Hub, with suggestions on Availability Zone setup, Estimated cost, Auto scaling group, etc.

Please note that for the sake of simplicity, this lab application utilises a monolithic architecture instead of distributed one. So in practice, you can expect those distributed web application will receive more comprehensive resiliency recommendations from the AWS Resilience Hub.
![Image of Yaktocat](https://github.com/JerryChenZeyun/Build-resilient-applications-using-AWS-Backup/blob/main/images/resiliency-recommendation.png)


#### Review of Best Practices Implemented

** You have utilised **AWS Resilience Hub** to automatically collect application information through the Cloudformation stack used to implement the application.
** You have defined a policy to define RTO/RPO requirements for the specific application
** **AWS Resilience Hub** conducts automatic assessment to generate resiliency report, with the assessment data to showcase if the defined RTO/RPO requirements have been met, alongside relevant optimization suggestions to achieve better application resiliency. 


### Step6: Tear Down

The following instructions will remove the resources that you have created in this lab.

#### Cleaning up AWS Resilience Hub Resources

1. Sign in to the AWS Management Console and navigate to the AWS Backup console - <br />https://us-east-1.console.aws.amazon.com/resiliencehub/home?region=us-east-1#/dashboard

2. Click on **Applications** and select the **WEB-WA-APP**, then click **Delete** under the **Actions** button.

3. Type in **Delete** in the pop up window to proceed.

4. Click on **Policies**, select **WA-LAB-RESILIENCY-POLICY**, then click **Delete** under the **Actions** button.

5. Type in **Delete** in the pop up window to proceed.

#### Cleaning up AWS Backup Resources

1. Sign in to the AWS Management Console and navigate to the AWS Backup console - <br />https://us-east-1.console.aws.amazon.com/backup/home?region=us-east-1#home

2. Click on **BACKUP VAULTS** from the menu on the left side, and select **BACKUP-LAB-VAULT**.

3. Under the section **Recovery points**, delete all the RECOVERY POINTS.

4. Once all the **RECOVERY POINTS** have been deleted, delete the **Backup Vault** by clicking on **DELETE VAULT** on the top right hand corner.

5. Click on **BACKUP PLANS** from the menu on the left side, and select **BACKUP-LAB**.

6. Scroll down to the section **RESOURCE ASSIGNMENTS**, and delete the resource assignment.

7. Delete the **BACKUP PLAN** by clicking on **DELETE** on the upper right corner of the screen.

#### Cleaning up the CloudFormation Stack

1. Sign in to the AWS Management Console and navigate to the AWS CloudFormation console -<br /> https://console.aws.amazon.com/cloudformation/

2. Select the stack **WA-Backup-Lab**, and delete the stack.

#### Cleaning up the CloudWatch Logs

1. Sign in to the AWS Management Console, and open the CloudWatch console at https://console.aws.amazon.com/cloudwatch/ .

2. Click **Logs** in the left navigation, and then select **Log groups**.

3. Click the check box on the left of the **/aws/lambda/RestoreTestFunction**.

4. Click the **Actions Button** then click **Delete Log Group**.

5. Verify the log group name then click **Delete**.


## Survey

Let us know what you thought of this lab and how we can improve the experience for you in the future by completing [this session poll](https://amazonmr.au1.qualtrics.com/jfe/form/SV_ehwTCMiRy46skbY?Session=HOL005). Participants who complete the surveys from AWS Innovate - Modern Applications Edition will receive a gift code for USD25 in AWS credits1, 2 & 3. AWS credits will be sent via email by November 30, 2022.

**Note:** Only registrants of AWS Innovate - Modern Applications Edition who complete the surveys will receive a gift code for USD25 in AWS credits via email.

1. AWS Promotional Credits Terms and conditions apply: https://aws.amazon.com/awscredits/
2. Limited to 1 x USD25 AWS credits per participant.
3. Participants will be required to provide their business email addresses to receive the gift code for AWS credits.





