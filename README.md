## Security orchestration, automation, and response (SOAR) with AWS Security Hub

This repo contains the CloudFormation template to deploy Security Hub custom actions, CloudWatch Event Rules and Lambda functions as detailed in the AWS Security Blog Post: Security orchestration, automation, and response (SOAR) with AWS Security Hub.

***QUICK NOTE:*** The end-to-end combination of a Security Hub custom action, a CloudWatch Event rule, a Lambda function, plus any supporting services needed to perform a specific action is referred to as a “playbook.”

### Get Started

Download the AWS CloudFormation template (`SecurityHubSOAR_CloudFormation.yaml`) and deploy it from the console

### Solutions Architecture
![Architecture](https://github.com/aws-samples/aws-security-hub-remediation-code/blob/master/Architecture.jpg)
1.	Integrated services send their findings to Security Hub.
2.	From the Security Hub console, you’ll choose a custom action for a finding. Each custom action is then emitted as a CloudWatch Event.
3.	The CloudWatch Event rule triggers a Lambda function. This function is mapped to a custom action, based on the custom action’s ARN.
4.	Dependent on the particular rule, the Lambda function  invoked will perform a remediation action on your behalf

### What remediation playbooks are covered and how?
All playbooks are made up of CloudWatch Events & Lambda functions

#### CIS AWS Benchmark Controls: ####
-	**1.3 – “Ensure credentials unused for 90 days or greater are disabled”**
-	**1.4 – “Ensure access keys are rotated every 90 days or less”**
    - A Lambda function will loop through the User's access key as identified in the finding. Any keys over 90 days are deleted
-	**1.5 – “Ensure IAM password policy requires at least one uppercase letter”**
-	**1.6 – “Ensure IAM password policy requires at least one lowercase letter”**
-	**1.7 – “Ensure IAM password policy requires at least one symbol”**
-	**1.8 – “Ensure IAM password policy requires at least one number”**
-	**1.9 – “Ensure IAM password policy requires a minimum length of 14 or greater”**
-	**1.10 – “Ensure IAM password policy prevents password reuse”**
-	**1.11 – “Ensure IAM password policy expires passwords within 90 days or less”**
    - A Lambda function will call the IAM UpdateAccountPasswordPolicy API with CIS-compliant parameters for the password policy
-	**2.2 – “Ensure CloudTrail log file validation is enabled”**
    - A Lambda function will parse out the CloudTrail information from the finding and call the UpdateTrail API to turn log file validation back on
-	**2.3 – “Ensure the S3 bucket CloudTrail logs to is not publicly accessible”**
    - A Lambda function will parse out the S3 Bucket information from the finding and calls the Systems Manager StartAutomationExecution API to run the Automation document `AWS-DisableS3BucketPublicReadWrite` to remove public Read & Write access from the bucket
-	**2.4 – “Ensure CloudTrail trails are integrated with Amazon CloudWatch Logs”**
    - To automatically ensure your CloudTrail logs are sent to CloudWatch, the Lambda function for this playbook will create a brand new CloudWatch Logs group that has the name of the non-compliant CloudTrail trail in it for easy identification. The Lambda function will programmatically update your non-compliant CloudTrail trail to send its logs to the newly created log group.  To accomplish this, CloudTrail needs an IAM role and permissions to be allowed to publish logs to CloudWatch. To avoid creating multiple new IAM roles and policies via Lambda, you’ll populate the ARN of this IAM role in the Lambda environmental variables for this playbook.
-	**2.6 – “Ensure S3 bucket access logging is enabled on the CloudTrail S3 bucket”**
    - To ensure the S3 bucket that contains your CloudTrail logs has access logging enabled, the Lambda function for this playbook invokes the Systems Manager document `AWS-ConfigureS3BucketLogging` this document will enable access logging for that bucket. To avoid statically populating your S3 access logging bucket in the Lambda function’s code, you’ll pass that value in via an environmental variable.
-	**2.7 – “Ensure CloudTrail logs are encrypted at rest using AWS KMS CMKs”**
    - The Code and Instructions are provided in the Blog post
-	**2.8 – “Ensure rotation for customer created CMKs is enabled”**
    - A Lambda function will parse out the KMS CMK infromation from the finding and call the KMS EnableKeyRotation API to enable rotation
-	**2.9 – “Ensure VPC flow logging is enabled in all VPCs”**
    - To enable VPC flow logging for rejected packets, the Lambda function for this playbook will create a new CloudWatch Logs group. For easy identification, the name of the group will include the non-compliant VPC name. The Lambda function will programmatically update your VPC to enable flow logs to be sent to the newly created log group. Similar to CloudTrail logging, VPC flow log need an IAM role and permissions to be allowed to publish logs to CloudWatch. To avoid creating multiple new IAM roles and policies via Lambda, you’ll populate the ARN of this IAM role in the Lambda environmental variables for this playbook.
-	**4.1 – “Ensure no security groups allow ingress from 0.0.0.0/0 to port 22”**
-	**4.2 – “Ensure no security groups allow ingress from 0.0.0.0/0 to port 3389”**
    - A Lambda function will parse out the Security Group information from the finding and calls the Systems Manager StartAutomationExecution API to run the Automation document `AWS-DisablePublicAccessForSecurityGroup` to remove 0.0.0.0/0 rules from the Security Group


#### Custom Playbooks
- **Send Findings to JIRA**
    - A Lambda function calls the Systems Manager StartAutomationExecution API to run the Automation document `AWS-CreateJiraIssue` JIRA information is provided via Lambda env vars
- **Apply Patch Baseline**
    - A Lambda function calls the Systems Manager SendCommand API to invoke the `AWS-UpdateSSMAgent` and `AWS-RunPatchBaseline` Documents on the instance. **NOTE**: This Playbook must be applied from Inspector vulnerability findings only!

### How do I perform response and remediation automatically?
You can modify your CloudWatch Event to use the `Security Hub Findings - Imported` **detail-type** and specify the Title of the specific CIS Finding as shown below. When findings that match this pattern are encountered, they will invoke your Lambda function without having to specify a Custom Action in Security Hub.

```
 {
   "source": [
     "aws.securityhub"
   ],
   "detail-type": [
     "Security Hub Findings - Imported"
   ],
   "detail": {
     "findings": {
      "Title": [
         "2.9 Ensure VPC flow logging is enabled in all VPCs"
      ]
     }
   }
 }
```

### How can I use these playbooks from my Security Hub Master Account?
You can add an IAM Policy to your Lambda function's execution role that allows a Role in the Master Account to assume those Lambda functions. 

For more information see this Premium Support Knowledge Center post on cross-account Lambda policies: https://aws.amazon.com/premiumsupport/knowledge-center/lambda-function-assume-iam-role/

## License

This library is licensed under the MIT-0 License. See the LICENSE file.