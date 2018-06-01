# Enabling Rotation for an Amazon RDS Database Secret<a name="enable-rotation-rds"></a>

You can enable rotation for a secret that has credentials for a [supported Amazon RDS database](rotating-secrets-rds.md#rds-supported-database-list) by using the [AWS Secrets Manager console](#proc-enable-rotation-rds-console), the AWS CLI, or one of the AWS SDKs\.

**Warning**  
Enabling rotation causes the secret to rotate once immediately when you save the secret\. Before you enable rotation, ensure that all of your applications that use this secret's credentials are updated to retrieve the secret from Secrets Manager\. The original credentials might not be usable after the initial rotation\. Any applications that you fail to update break as soon as the old credentials are no longer valid\.

**To enable and configure rotation for a supported Amazon RDS database secret**  
Follow the steps under one of the following tabs:

------
#### [ Using the AWS Management Console ]<a name="proc-enable-rotation-rds-console"></a>

**Minimum permissions**  
To enable and configure rotation in the console, you must have the permissions that are provided by the following managed policies:  
`SecretsManagerReadWrite` – Provides all of the Secrets Manager, Lambda, and AWS CloudFormation permissions\.
`IAMFullAccess` – Provides the IAM permissions that are required to create a role and attach a permission policy to it\.

1. Sign in to the AWS Secrets Manager console at [https://console\.aws\.amazon\.com/secretsmanager/](https://console.aws.amazon.com/secretsmanager/)\.

1. Choose the name of the secret that you want to enable rotation for\.

1. In the **Configure automatic rotation** section, choose **Enable automatic rotation**\. This enables the other controls in this section\.

1. For **Select rotation interval**, choose one of the predefined values—or choose **Custom**, and then type the number of days you want between rotations\. If you're rotating your secret to meet compliance requirements, then we recommend that you set this value to at least 1 day less than the compliance\-mandated interval\. 
**Note**  
If you use the Lambda function that's provided by Secrets Manager to alternate between two users \(the console uses this template if you choose the second "master secret" option in the next step\), then you should set your rotation period to one\-half of your compliance\-specified minimum interval\. This is because the old credentials are still available \(if not actively used\) for one additional rotation cycle\. The old credentials are fully invalidated only after the user is updated with a new password after the second rotation\.   
If you modify the rotation function to immediately invalidate the old credentials after the new secret becomes active, then you can extend the rotation interval to your full compliance\-mandated minimum\. Leaving the old credentials active for one additional cycle with the `AWSPREVIOUS` staging label provides a "last known good" set of credentials that you can use for fast recovery\. If something happens that breaks the current credentials, you can simply move the `AWSCURRENT` staging label to the version that has the `AWSPREVIOUS` label\. Then your customers should be able to access the resource again\. For more information, see [Rotating AWS Secrets Manager Secrets by Alternating Between Two Existing Users](rotating-secrets-two-users.md)\.

1. Specify the secret with credentials that the rotation function can use to update the credentials on the protected database\.<a name="considerations-for-rotation"></a>
   + **Use this secret**: Choose this option if the credentials in this secret have permission in the database to change their own password\. Choosing this option causes Secrets Manager to implement a Lambda function that rotates secrets with a single user that gets its password changed with each rotation\.
**Note**  
This option is the "lower availability" option\. This is because sign\-in failures can occur between the moment when the old password is removed by the rotation and the moment when the updated password is made accessible as a new version of the secret\. This time window should be very short, on the order of a few seconds or less, but it can happen\.   
If you choose the **Use this secret** option, ensure that your client apps that sign in with the secret use an appropriate ["backoff and retry with jitter" strategy](http://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) in code\. A real failure should be reported only if signing in fails several times over a longer period of time\.
   + **Use a secret that I have previously stored in AWS Secrets Manager**: Choose this option if the credentials in the current secret don't have permissions to update the credentials, or you require high availability for the secret\. To choose this option, you must create a separate "master" secret with credentials that have permissions to update the current secret's credentials\. Then choose the master secret from the list\. Choosing this option causes Secrets Manager to implement a Lambda function that rotates secrets by creating a new user and password with each rotation, and deprecating the old one\.
**Note**  
This is the "high availability" option because the old version of the secret continues to operate and handle service requests while the new version is prepared and tested\. The old version is not deleted until after the clients switch to the new version\. There's no downtime while changing between versions\.  
This option requires the Lambda function to clone the permissions of the original user and apply them to the new user in each rotation\.

1. Choose **Save** to store your changes and to trigger the initial rotation of the secret\.

1. If you chose to rotate your secret with a separate master secret, then you must manually grant your Lambda rotation function permission to access the master secret\. Follow these instructions:

   1. When rotation configuration completes, the following message appears at the top of the page:

      *Your secret *<secret name>* has been successfully stored and secret rotation is enabled\. To finish configuring rotation, you need to provide the *role* permissions to access the value of the secret *<ARN of your master secret>**\.

      You must manually modify the policy for the role to grant the rotation function GetSecretValue access to the master secret\. Secrets Manager can't do this for you for security reasons\. Rotation of the secret fails until you complete the following steps because it can't access the master secret\.

   1. Copy the Amazon Resource Name \(ARN\) from the message to your clipboard\.

   1. Choose the link on the word "role" in the message\. This opens the IAM console to the role details page for the role attached to the Lambda rotation function that Secrets Manager created for you\.

   1. On the **Permissions** tab, choose **Add inline policy**, and then set the following values:
      + For **Service**, choose **Secrets Manager**\.
      + For **Actions**, choose **GetSecretValue**\.
      + For **Resources**, choose **Add ARN** next to the **secret** resource type entry\.
      + In the **Add ARN\(s\)** dialog box, paste the ARN of the master secret that you copied previously\.

   1. Choose **Review policy**, and then choose **Create policy**\.
**Note**  
As an alternative to using the Visual Editor as described in the previous steps, you can paste the following statement into an existing or new policy:  

      ```
      {
          "Effect": "Allow",
          "Action": "secretsmanager:GetSecretValue",
          "Resource": "<ARN of the master secret>" 
      }
      ```

   1. Return to the AWS Secrets Manager console\.

If there's not already an ARN for a Lambda function that's assigned to the secret, Secrets Manager creates the function, assigns all required permissions, and configures it to work with your database\. Secrets Manager counts down the number of days specified in the rotation interval\. When it reaches zero, Secrets Manager rotates the secret again and resets the interval for the next cycle\. This continues until you disable rotation\.

------
#### [ Using the AWS CLI or SDK Operations ]

**Minimum permissions**  
To enable and configure rotation in the console, you must have the permissions that are provided by the following managed policies:  
`SecretsManagerReadWrite` – Provides all of the Secrets Manager, Lambda, and AWS CloudFormation permissions\.
`IAMFullAccess` – Provides the IAM permissions that are required to create a role and attach a permission policy to it\.

You can use the following Secrets Manager commands to configure rotation for an existing secret for a supported Amazon RDS database:
+ **API/SDK:** [RotateSecret](http://docs.aws.amazon.com/secretsmanager/latest/apireference/API_RotateSecret.html)
+ **AWS CLI:** [RotateSecret](http://docs.aws.amazon.com/cli/latest/reference/secretsmanager/rotate-secret.html)

You also need to use commands from AWS CloudFormation and AWS Lambda\. For more information about the commands that follow, see the documentation for those services\.

**To create a Lambda rotation function by using an AWS Serverless Application Repository template**  
The following is an example AWS CLI session that performs the equivalent of the console\-based rotation configuration that's described in the **Using the AWS Management Console** tab\. You create the function by using an AWS CloudFormation change set\. Then you configure the resulting function with the required permissions\. Finally, you configure the secret with the ARN of the completed function, and rotate once to test it\.

The following example uses the generic template, so it uses the last ARN that was shown earlier\.

If your database or service resides in a VPC provided by Amazon VPC, then you must include the fourth command below that configures the function to communicate with that VPC\. If no VPC is involved, then you can skip that command\.

The first command sets up an AWS CloudFormation change set based on the template provided by Secrets Manager\.

You use the `--application-id` parameter to specify which template to use\. The value is the ARN of the template\. For the list of templates provided by AWS and their ARNs, see [AWS Templates You Can Use to Create Lambda Rotation Functions ](reference_available-rotation-templates.md)\.

The templates also require additional parameters that are provided with `--parameter-overrides`, as shown in the example that follows\. This parameter is required to pass two pieces of information as Name and Value pairs to the template that affect how the rotation function is created:
+ **endpoint** – The URL of the service endpoint that you want the rotation function to query\. Typically, this is `https://secretsmanager.region.amazonaws.com`\.
+ **functionname** – The name of the completed Lambda rotation function that's created by this process\.

```
$ aws serverlessrepo create-cloud-formation-change-set \
          --application-id arn:aws:serverlessrepo:us-east-1:297356227824:applications/SecretsManagerRotationTemplate \
          --parameter-overrides '[{"Name":"endpoint","Value":"https://secretsmanager.region.amazonaws.com"},{"Name":"functionName","Value":"MyLambdaRotationFunction"}]' \
          --stack-name MyLambdaCreationStack
{
    "ApplicationId": "arn:aws:serverlessrepo:us-west-2:297356227824:applications/SecretsManagerRDSMySQLRotationSingleUser",
    "ChangeSetId": "arn:aws:cloudformation:us-west-2:123456789012:changeSet/EXAMPLE1-90ab-cdef-fedc-ba987EXAMPLE/EXAMPLE2-90ab-cdef-fedc-ba987EXAMPLE",
    "StackId": "arn:aws:cloudformation:us-west-2:123456789012:stack/aws-serverless-repository-MyLambdaCreationStack/EXAMPLE3-90ab-cdef-fedc-ba987EXAMPLE"
}
```

The next command runs the change set that you just created\. The change\-set\-name parameter comes from the `ChangeSetId` output of the previous command\. This command produces no output\.

```
$ aws cloudformation execute-change-set --change-set-name arn:aws:cloudformation:us-west-2:123456789012:changeSet/EXAMPLE1-90ab-cdef-fedc-ba987EXAMPLE/EXAMPLE2-90ab-cdef-fedc-ba987EXAMPLE
```

The following command grants the Secrets Manager service permission to call the function on your behalf\. The output shows the permission added to the role's trust policy\.

```
$ aws lambda add-permission \
          --function-name MyLambdaRotationFunction \
          --principal secretsmanager.amazonaws.com \
          --action lambda:InvokeFunction \
          --statement-id SecretsManagerAccess
{
    "Statement": "{\"Sid\":\"SecretsManagerAccess\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"secretsmanager.amazonaws.com\"},\"Action\":\"lambda:InvokeFunction\",\"Resource\":\"arn:aws:lambda:us-west-2:123456789012:function:MyLambdaRotationFunction\"}"
}
```

The following command is required only if your database is running in a VPC\. If it isn't, skip this command\. Look up the VPC information for your Amazon RDS instance by using either the Amazon RDS console, or by using the `aws rds describe-instances` CLI command\. Then put that information in the following command and run it\.

```
$ aws lambda update-function-configuration \
          --function-name arn:aws:lambda:us-west-2:123456789012:function:MyLambdaRotationFunction \
          --vpc-config SubnetIds=<COMMA SEPARATED LIST OF VPC SUBNET IDS>,SecurityGroupIds=<COMMA SEPARATED LIST OF SECURITY GROUP IDs>
```

If you created a function using a template that requires a master secret, then you must also add the following statement to the function's role policy\. For complete instructions, see [Granting a Rotation Function Permission to Access a Separate Master Secret](auth-and-access_identity-based-policies.md#permissions-grant-rotation-role-access-to-master-secret)\.

```
        {
             "Action": "secretsmanager:GetSecretValue",
             "Resource": "arn:aws:secretsmanager:region:123456789012:secret:MyDatabaseMasterSecret",
             "Effect": "Allow"
        },
```

Finally, you can apply the rotation configuration to your secret and perform the initial rotation\.

```
$ aws secretsmanager rotate-secret \
          --secret-id production/MyAwesomeAppSecret \
          --rotation-lambda-arn arn:aws:lambda:us-west-2:123456789012:function:aws-serverless-repository-SecretsManagerRDSMySQLRo-10WGBDAXQ6ZEH \
          --rotation-rules AutomaticallyAfterDays=7
```

We recommend that even if you want to create your own Lambda rotation function for a supported Amazon RDS database, you should follow the preceding steps that use the SecretsManagerRotationTemplate AWS CloudFormation template\. This is because it lets Secrets Manager set up most of the permissions and configuration settings for you\.

------