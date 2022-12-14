# Elevate API Security with Cognito and WAF


This lab is provided as part of **[AWS Innovate Modern Applications Edition](https://aws.amazon.com/events/aws-innovate/apj/modern-apps/)**. 

Click [here](https://github.com/phonghuule/aws-innovate-modern-applications-2022) to explore the full list of hands-on labs.

ℹ️ You will run this lab in your own AWS account. Please follow directions at the end of the lab to remove resources to avoid future costs.

## About this Lab

Digital businesses use public facing APIs to deliver values to their consumer. Because of that, security is a top concern for all organizations seeking to develop APIs supporting their business model. 

Application leaders must create and implement effective API security strategy that aligns with their business needs. An example of this is the [Zero Trust](https://aws.amazon.com/blogs/publicsector/how-to-think-about-zero-trust-architectures-on-aws/) strategy that ensures only authorized requests are permitted to access the business layer of your application and trust evaluations to be performed at multiple layers of the architecture.

In this lab we will walk you through an example scenario of securing your API at multiple layers in AWS. We will gradually tighten the security of the API at each layer, using the following services:

* [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html) - Used for securing REST API.
* [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) - Used to securely store secrets.
* [Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) - Used to prevent direct access to API as well as to enforce encrypted end-to-end connections to origin.
* [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html) - Used to protect our API by filtering, monitoring, and blocking malicious traffic.
* [Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html) - Used to enable access control for API

**Note:** For simplicity, we have used North Virginia **'us-east-1'** as the default region for this lab. Please ensure all lab interaction is completed from this region.

### Content 

This lab is divided into several sections as follows:

- [Step 1. Deploy the lab base infrastructure](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF#step-1-Deploy-the-lab-base-infrastructure)
- [Step 2. Use secrets securely](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF#step-2-Use-secrets-securely)
- [Step 3. Prevent requests from accessing API directly](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF#step-3-Prevent-requests-from-accessing-API-directly)
- [Step 4. Application layer defense](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF#step-4-Application-layer-defense)
- [Step 5. Contol access to API](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF#step-5-Contol-access-to-API)
- [Teardown](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF#Teardown)
- [Survey](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF#Survey)


## Step 1. Deploy the lab base infrastructure

In this section we will build out our base lab infrastructure. This infrastructure consists of a public API gateway which connects to Lambda (application layer). The application layer will connect to RDS for MySQL (database layer) within a [Virtual Private Cloud (VPC)](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html). The environment will be deployed to separate private subnets which will allow for segregation of application and network traffic across multiple [Availability Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html). We will also deploy an [Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) and [NAT gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) along with appropriate routes from both public and private subnets.


When we successfully complete our initial stage template deployment, our deployed workload should reflect the following diagram:

![Section1 Base Architecture](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-deploy_the_lab_base_infrastructure.png)

Note the following:

1. The API Gateway has been provided with a role to allow access to invoke the Lambda function in the private subnet (application layer). 

2. The Lambda function has been provided with a role to allow the API Gateway to invoke the Lambda function.

3. Secrets Manager has been configured as the main password store which the Lambda function will retrieve to provide access to RDS. This will allow Secrets Manager to be used to encrypt, store and transparently decrypt the password when required.

4. The Security Group associated with Amazon RDS for MySQL will **only** allow inbound traffic on port 3306 from the specific security group associated with Lambda. This will allow sufficient access for Lambda to connect to Amazon RDS for MySQL. 



**Note:** For simplicity, we have used North Virginia **'us-east-1'** as the default region for this lab. Please ensure all lab interaction is completed from this region.


To deploy the template for the base infrastructure complete the following steps:

### 1.1. Deploy Cloudformation Template.



1. Download the CloudFormation template [here.](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/raw/main/Code/templates/section1/section1-base.yaml "Section1 template"), then sign in to the AWS Management Console as an IAM user and open the S3 console at [console.aws.amazon.com/s3](https://console.aws.amazon.com/s3) as shown:

![Section1 S3Bucket](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-s3bucket.png?raw=true)


2. Create a bucket with a unique name and select **us-east-1(N.Virginia)** as the region as shown:

![Section1 Create S3Bucket](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-create_s3bucket.png)

3. Take note of your **S3 Bucket name**, which we will need later in the lab when we create a Cloudformation stack:

![Section1 S3Bucket Available](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-s3_bucket_available.png)

4. Download our 3 Lambda deployment packages:

* [rds-create-table.zip](https://d3h9zoi3eqyz7s.cloudfront.net/Security/300_multilayer_api_security_with_congnito_and_waf/rds-create-table.zip "Section1 rds-create-table.zip")
* [rds-query.zip](https://d3h9zoi3eqyz7s.cloudfront.net/Security/300_multilayer_api_security_with_congnito_and_waf/rds-query.zip "Section1 rds-query.zip") 
* [python-requests-lambda-layer.zip](https://d3h9zoi3eqyz7s.cloudfront.net/Security/300_multilayer_api_security_with_congnito_and_waf/python-requests-lambda-layer.zip "Section1 python-requests-lambda-layer.zip")

These packages will be used to build the lambda environment later in the lab.

When you have downloaded all packages, upload all 3 to your new bucket with the sames filesnames:

![Section1 Lambda packages](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-upload_lambda_packages.png)

5. Open the CloudFormation console at [console.aws.amazon.com/cloudformation](https://console.aws.amazon.com/cloudformation/) and select **us-east-1(N.Virginia)** as your AWS Region:

![Section1 Select Region](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-select_region.png)

6. Select the stack template which you downloaded earlier, and create a stack:

![Section1 Create Stack](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-create_stack.png)

7. For the stack name use **'walab-api'** and enter the bucket name you just created in the parameters. You can leave all other parameters as default value. When you are ready, click the 'Next' button. 

![Section1 Add API Gateway URL](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-default_stack_s3bucket.png)

8. Scroll down to the bottom of the stack creation page and acknowledge the IAM resources creation by selecting **all the check boxes**. Then launch the stack. It may take 9~10 minutes to complete this deployment.

![Section1 Acknowledge IAM resources creation](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-create-IAM-resources.png)

9. Go to the **Outputs** section of the cloudformation stack, click **Cloud9URL** to set up your test environment.

![Section1 Cloud9](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-cloud9.png)

10. Run `cd walab-scripts` to ensure Cloud9 automatically cloned all scripts.

![Section1 git clone](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-git_clone_complete.png)

11. Run `bash install_package.sh`
* this script will install boto3 and requests

![Section1 Install python packages](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-install_python-packages.png)



### 1.2. Confirm Successful Application Deployment.

1. Go to the **Outputs** section of the cloudformation stack you just deployed and copy **APIGatewayURL** to make sure if the lab base infrastruture has been successfully deployed.

![Section1 Access Data using API](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-access_data_using_API.png)


Take a note of APIGatewayURL as we will often use this URL for testing.


2. In Cloud9, execute the script called sendRequest.py with the argument of your APIGatewayURL.
```
python sendRequest.py 'APIGatewayURL'
```
Once your command runs successfully, you should be seeing Response code 200 with Response data as shown here:


![Section1 Test API Cloud9](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-test_api_cloud9.png)

### Step 2. Use secrets securely

Passwords remain vulnerable to brute force attack methods even if we store our secrets in a secure way. Because of this we can augment our deployed architecture to limit the lifespan of a password through the use of automatic rotation. An ideal way to approach this task is through the use of AWS Secrets Manager, which can enable you to automatically rotate secrets for other databases or 3rd party services.

The second section of the lab will demonstrate how to configure AWS Secrets Manager to securely store your database credentials and automatically rotate them on a schedule for our deployed database. Follow these steps to complete the configuration:

### 2.1. Access Secrets Manager from the Console.

1. Go to the **Outputs** section of the CloudFormation stack you just deployed and click **RDSMysqlSecret** as shown. This will allow you to view AWS Secrets Manager from the console. Alternatively, you can launch the service from the console main menu in the usual way.


![Section2 AWS Secrets Manager](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-secrets_manager.png)

2. The secret name will begin with **WARDSInstanceRotationSecret** as shown.


![Section2 RDS Secret](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-rds_secret.png)

3. By default, AWS Secrets Manager uses [AWS KMS](https://docs.aws.amazon.com/secretsmanager/latest/userguide/services-secrets-manager.html) to encrypt as well as to decrypt your secrets. You can see all current secret values by clicking on the secret name and then selecting **Retrieve secret value** from the dialog box as shown:


![Section2 Retrieve secret value](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-retrieve_secret_value.png)

4. As shown below, all sentitive information such as username, password, and RDS endpoint are stored as part of the encrypted secret value. Take this opportunity to record the current password value which we will rotate later in the lab.


![Section2 Current Credentials](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-current_credentials.png)

### 2.2. Configure Password Rotation.

AWS Secrets Manager can be configured to automatically rotate a password for an Amazon RDS database. When you enable rotation for Amazon RDS, Secrets Manager provides a complete, ready-to-run Lambda rotation function. More information on this topic can be found [here.](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets-rds.html).

1. To enable rotation, choose **Edit rotation**.


![Section2 Rotate Secret](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-rotate_secret.png)

2. Choose **Enable automatic rotation** and select the rotation interval based on your requirement.
    
We will **create a new Lambda function** designed for RDS MySQL. Once you define the new AWS Lambda function name, choose **Use this secret** at the bottom of the dialog box which will select the secret information which will be subjected to rotation as shown.


![Section2 Create Lambda for rotation](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-create_lambda.png)

Note that the enabling process may take a few minutes so please be patient.

3. Now that automated rotation has been enabled, click **Rotate secret immediately** to test the rotation as shown. Note that intially **this step will fail**, so remain calm as we will fix this later on
!


![Section2 Enabled rotation](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-enabled_rotation.png)

4. As mentioned previously, rotation will fail. This is because the new Lambda rotation function created by Secrets Manager is not associated with our Lambda Security Group that we explicitly allowed to connect to Amazon RDS for MySQL via port 3306. In order for our newly created rotation to work, your network environment must permit the Lambda rotation function to communicate with your database and the Secrets Manager service. More details on this topic can be found [here](https://aws.amazon.com/premiumsupport/knowledge-center/rotate-secrets-manager-secret-vpc/)


![Section2 Failed to rotate](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-failed_to_rotate.png)

5. A simple workaround to correct our misconfigured rotation is to replace the **Security Group** and **Subnet** in new our Lambda rotation function. To complete this, access **Lambda Function** in AWS console and choose **SecretManager-rds-mysql-rotation**. Note that the function name will be the one you defined at step 2 above as shown.


![Section2 Secrets manager lambda](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-secrets-manager-lambda.png)

6. Go to the **Configuration** section and **VPC** on the left panel. You can see it's associated with all RDS subnet and security group.


![Section2 Lambda with RDS SG](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-lambda_with_rds_sg.png)

7. Choose **Edit** and remove all Subnet and Security Groups. Then search for **WAprivateLambdaSubnet** for Subnets (select 2 for each availability zone) and **LambdaSecurityGroup** for the Security Group to add them. Confirm that your dialog box looks similar to the screenshot shown below and click **Save**. Note that this process sometimes takes 2~3 minutes to be completed, so please be patient.


![Section2 Edit Lambda Subnet SG](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-edit_lambda_subnet_sg.png)

8. Our new Lambda rotation function is now associated with **LambdaSecurityGroup** allowed to connect to RDS for MySQL. Your screen should look similar to the screenshot below:


![Section2 Lambda with Lambda SG](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-lambda_with_lambda_sg.png)

9. Now we should attempt to successfully rotate the password again. To do this, go back to AWS Secrets Manager and select **Rotate secret immediately** to test the rotation. You should now see a message informing you of a successful rotation as shown:


![Section2 Completed rotation](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-completed_rotation.png)

If you are experiencing the same failure as previously, check that you have added the correct **Security Group** and **Subnet** information and retry. This process sometimes takes a few minutes to complete so please be patient.

10. Inspecting the password data within Secrets Manager should now show a change to the original password information which you recorded earlier.


![Section2 New Credentials](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section2/section2-new_secret.png)

11. Let's confirm if the password in the database has been successfully updated based on the automatic rotation. To do this, go back to **Cloud9** in the console and execute the script called sendRequest.py with the argument of your APIGatewayURL. You will need to change to the walab-scripts directory to execute the script. Make sure you replace **'APIGatewayURL'** with the value you previously used from the Cloudformation stack output.
```
python sendRequest.py 'Enter your APIGatewayURL here'
```
Once your command runs successfully, you should be see a 200 Response code with Response data as shown:


![Section1 Test API Cloud9](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-test_api_cloud9.png)


### Step 3. Prevent requests from accessing API directly


In this section you will be building your own distribution of [Amazon CloudFront](https://aws.amazon.com/cloudfront/) which offers protection at the edge of your architecture. When integrating CloudFront with regional API endpoints, not only does the service distribute traffic across multiple edge locations to improve performance, but also it supports geoblocking, which you can use to help block requests from particular geographic locations from being served. With Amazon CloudFront, you can also enforce encrypted end-to-end connections to an origin API by using HTTPS. Additionally, CloudFront can automatically close connections from slow reading or slow writing attackers. API Gateway can then be configured to **only** accept requests only from CloudFront. This helps prevent anyone from accessing your API Gateway deployment directly. In this section of the lab, you will use a CloudFront custom header value generated by AWS Secrets Manager in combination with [AWS WAF](https://aws.amazon.com/waf/). 

This will build out the architecture as follows:


![Section3 Security Enhanced Architecture](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-security_enhanced_architecture.png)

To complete this section, follow the steps listed below:

### 3.1. Deploy Security Enhancement.

1. Download the CloudFormation template [here](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/raw/main/Code/templates/section3/section3-enhance_security.yaml "Section1 template"), then sign in to the AWS Management Console as an IAM user or role in your AWS account, and open the CloudFormation console at [console.aws.amazon.com/cloudformation](https://console.aws.amazon.com/cloudformation/).

2. Choose a stack template.

![Section3 Create Stack](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-create_stack.png)

3. Go to the **Outputs** section of **your previous cloudformation stack** that you used to deploy the base lab infrastructure. Copy and paste **APIGatewayURL** and provide your **S3 bucket name**. The other values can be left as default. When you are finished with the configuration click 'Next' as shown below.
It may take 3~4 minutes to complete this deployment.

![Section3 Add API Gateway URL S3Bucket](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-add_api_url_S3bucket.png)

4. Review the template and scroll to the bottom of the screen. Make sure you acknowledge the IAM resource creation and Auto_expand capability by selecting the three tick boxes as shown. When you are ready, click on **create stack**.


![Section3 Acknowledge IAM resources creation](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section1/section1-create-IAM-resources.png)



### 3.2 Add custom HTTP headers to the requests.

When your stack has finished deployment, you can now add a custom header to the request in CloudFront. As previously described, this will prevent users from bypassing CloudFront to access your API directly. More information on this topic can be found [here.](https://aws.amazon.com/blogs/security/how-to-enhance-amazon-cloudfront-origin-security-with-aws-waf-and-aws-secrets-manager/)

To complete this configuration, complete the following steps:

1. When the current CloudFormation stack has finished deployment, go to the **Outputs** section of the stack. Verify that your **OriginVerifyHeaderName** is set to **X-Origin-Verify**. Now click the value for **OriginVerifyHeader** to take you back to AWS Secrets Manager as shown:


![Section3 Output OriginVerifyHeader](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-get_output_OriginVerifyHeader.png)

2. From AWS Secrets Manager, click the **OriginVerifyHeader** secret to get the **X-Origin-Verify** header value:


![Section3 Select Secret for Header Value](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-select_secret_for_header_value.png)

3. From the Secret Value dialog box, click **Retrieve secret value**. Record the **Secret Value** of HEADERVALUE. We will use this value as the **Origin Header** value in CloudFront. Secrets Manager will now be responsible for automatically rotating this header value to prevent possible future compromise in the same way that we rotated the password information earlier in the lab.


![Section3 Get Header Value](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-get_header_value.png)

4. Now we can configure CloudFront. You do this by either lauching CloudFront from the console in the standard way, or by clicking the URL of **CloudFrontConsole** in the **Outputs** section of the current CloudFormation stack. You should be able to see our newly created CloudFront distribution. Click on the **ID** field as shown:


![Section3 CloudFront Console](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-cloudfront_console.png)

5. Go to the **Origins and Origin Groups** tab and select your CloudFront Origin to edit its configuration as shown:


![Section3 Origins and Origin Gourps](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-origins_origin_groups.png)

6. In the **Origin Custom Headers** section, enter **'X-Origin-Verify'** as the **Header Name** and use the previously recorded value from the AWS Secrets Manager **HEADERVALUE** as the **Value** as shown:


![Section3 Add Custom Header and Value](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-add_header_value.png)

CloudFront will now add the **Origin Custom Header** into all incoming request from the client applications or user traffic.

7. Note that your configuration won't take effect until the distribution has propagated to the AWS edge locations. It may take 3~5 minutes so please be patient. You can check the status of the propagation as shown:


![Section3 Update CloudFront](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-update_cloudfront.png)

8. When your CloudFront propagation completes, we will move to configuring WAF. The second CloudFormation template which we deployed, automatically creates Web ACLs and added associated rules to inspect the incoming request header **X-Origin-Verify**. A Web ACL (Web Access Control List) is the core resource in an AWS WAF deployment. It contains rules that are evaluated for each request that it receives.

To review the WAF rules, go to the current CloudFormation stack **Outputs** tab and click **WAFWebACLR** to go to **Web ACLs** as shown:


![Section3 Output WAF](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-output_waf.png)

9. Go to **Rules**, there will be only one rule created as **{Your Stack Name}-XRule**.

![Section3 Rules in ACL](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-rules_in_acl.png)

10. By default, any requests that do not match the listed rules in the Web ACLs will be **blocked**. In this case, the rule will **allow** the request **ONLY** if the string value of **X-Origin-Verify** is valid. If the header isn’t valid, AWS WAF blocks the request:


![Section3 Verify Rules](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-verify_rules.png)

11. Let's test our configuration to check functionality. To do this go to the stack **Outputs** tab of the current CloudFormation template. From there, click **CloudFrontEndpoint** to verify if now API clients access your API only through a CloudFront distribution. It will automatically have query string.


![Section3 CloudFront Domain Name](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-domain_name.png)

Clicking on the link should show a successful query of your favourite player.


Take a note of CloudFrontEndpoint as we will often use this URL for testing.


## 3.3 Verify if API only accepts the request from CloudFront.

1. From the console access the  **Cloud9** service and go to your IDE. We will now test with both the **CloudFrontEndpoint** and the **APIGatewayURL** to see the difference in behaviour.

* Firstly, change to the **walab-script** directory and execute the script called sendRequest.py with the argument of your **CloudFrontEndpoint** as shown:

```
python sendRequest.py 'CloudFrontEndpoint'
```
* Note the output from the previous command and now execute the same script with the argument of your **APIGatewayURL** as shown:

```
python sendRequest.py 'APIGatewayURL'
```

You should notice a different output based on the argument that you pass to the script. When you run the script with the **CloudFrontEndPoint** argument, you should receive a 200 response code, which indicates a successful query. However, when you run the same script against the **APIGatewayURL** directly, you will notice that a 403 response code is generated. This indicates that the request is detected as forbidden as this request does not have the **X-Origin-Verify** field together with the required value. Run both script outputs a few times to generate some data.

Check that your output is similar to the screenshot below:


![Section3 Test API Cloud9](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-test_api_cloud9.png)

2. Now go back to the WAF Console and select the **Overview** tab to see the metrics of the ACL. You can confirm your request was blocked by WAF from this metrics. Click **{Your Stack Name}-ACLMetricR BlockedRequests** BlockedRequests to see only blocked request by SQL database. 


![Section3 ACLMetricR Blocked Requests](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section3/section3-metricR_blocked_requests.png)


Your graph shape will be different depending on the number of times you executed the script with the CloudFront Endpoint and the APIgateway address.


### Step 4. Application layer defense


In this section we will tighten security using AWS WAF further to mitigate the risk of vulnerabilities such as SQL Injection, Distributed denial of service (DDoS) and other common attacks. WAF allows you to create your own custom rules to decide whether to block or allow HTTP requests before they reach your application.

### 4.1. Identify the risk of vulnerabilities.

A SQL Injection attack consists of insertion of a SQL query via the input data to the application. A successful SQL injection exploit can be capable or reading sensitive data from a database, or in extreme cases data modification/deletion. 

Our current API retrieves data from RDS for MySQL and relies on the user interacting via CloudFront. However, it is still possible for malicious SQL code to be injected into a web request, which could result in unwanted data extraction.

As a simple example, simply add **'or 1=1'** at the end of your CloudFront domain name as shown:

```
Your_CloudFront_Domain_Name/?id=1 or 1=1
```


As you can see from the output, using this simple SQL injection could result in an attacker gaining access to all the data in our database:


![Section4 Access through CloudFront](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section4/section4-sql_injection.png)

This section of the lab will focus on some techniques which work to block web requests that contain malicious SQL code or SQL injection using AWS WAF.

### 4.2. Working with SQL injection match conditions.

1. From CloudFormation, locate the second stack which you deployed. On the stack **Outputs** tab, click **WAFWebACLG** to go to **Web ACLs** to review Rules in WAF. This web ACL is associated with CloudFront.


![Section4 Output WAF](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section4/section4-waf_global_cloudfront.png)

2. Go to the **Rules** tab and select **add managed rule groups** as shown:


![Section4 Add AWS Managed Rule](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section4/section4-add_aws_managed_rule.png)

3. Expand the **AWS managed rule groups** section and enable the **SQL database** rules as shown: 

![Section4 Enable SQL database](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section4/section4-enable_sql_database.png)

By doing this we are adding rules that allow you to block request patterns associated with exploitation specific to SQL databases, such as SQL injection attacks. Make sure you select **Add rules** at the bottom of the screen to proceed to the next stage.

4. It is possible to assign priorities based on the rules which you specify. As you only have one rule at this moment, we can skip this configuration. Click **Save**:


![Section4 Set Rule Priority](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section4/section4-set_rule_priority.png)

5. You should now be able to see the **AWS-AWSManagedRulesSQLiRuleSet** added to web ACL as shown:


![Section4 AWS AWSManagedRulesSQLiRuleSet](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section4/section4-AWS-AWSManagedRulesSQLiRuleSet.png)

6. We can now rerun our test again, which should hopefully produce a different output. 

Use your **CloudFrontEndpoint** to run the same query as before, inclusive of the injection attack at the end. This can be done in either a web-browser or your Cloud9 IDE environment using the script that we have provided previously:


![Section4 Block SQL Injection](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section4/section4-block_sql_injection.png)

If your configuration is correct, you should now see a **Response code: 403**. This means that WAF has **blocked** this request as malicious code has been detected in the input.


7. We can now check that the blocked request has registered in our ACL metrics. To do this, go to the **Overview** in WAF console to see the metrics of ACL. You can confirm your request was blocked by WAF from this metrics. Click **AWS-AWSManagedRulesSQLiRuleSet BlockedRequests** to see only blocked request by SQL database. Note that your output may differ from the screenshot below depending on the amount of blocked requests that you sent.


![Section4 SQL Injection MetricGblocked Requests](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section4/section4-sql_injection_metricGblocked_requests.png)


### Step 5. Contol access to API



In this section we will be building API Access Control with [Amazon Cognito](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-integrate-with-cognito.html). This will extend our architecture to ensure that only identified users are permitted access to the API.

### 5.1. Identify the risk of vulnerabilities.
Even though we have controlled traffic at multiple layers, anyone who knows your CloudFront Domain Name can access your API. Furthermore we do not know who accessed your API, so the owner of the traffic remains anonymous. Ideally we should ensure that only legitimate users who we are aware of are permitted access. This will mean that a successful API call will have to be both identified as well as authorized.

In this lab we will build out our architecture using Amazon Cognito. This addition will allow a user to sign in to a user pool which we create, obtain an identity or access token and then call the API method with one of the tokens. These tokens are typically set to the request's Authorization header.

### 5.2. Sign up with Amazon Cognito user pools.

The second CloudFormation template which you have already deployed already contains an Amazon Cognito User Pool. We will start by obtaining the **App client secret** of Cognito, which we will use to generate an ID token at later points in the lab.

<details>
<summary>Click here for for SignUp instructions for Amazon Cognito</summary>

1. Go to the **Outputs** section of the current cloudformation stack and locate the **CognitoSignupURL**. Click on the link as shown:


![Section5 Output Cognito](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-output_cognito.png)

2. Select the **Sign up** link with Cognito user pools.

![Section5 Sign up](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-sign_up.png)

3. Provide your valid email address and password. You will receive an email from **<no-reply@verificationemail.com>** with a link to confirm your email address is valid.

![Section5 Provide Username Password](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-provide_username_password.png)

4. Once you click the link in email, you will see the following confirmation.

![Section5 Confirmed Registration](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-confirmed_registration.png)

5. Go to **Amazon Cognito** in AWS Console and click **Manage User Pools**

![Section5 Cognito Console](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-cognito_console.png)

6. Select your user pools.

![Section5 Output WAF](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-select_user_pools.png)

7. Select **Users and groups** under **General settings** in the left panel. Note that your account status should now be shown as **CONFIRMED**:


![Section5 Output WAF](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-check_account_statue.png)

8. Select **App clients** under **General settings** in the left panel and click **Show Details**.

![Section5 Output WAF](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-select_app_client.png)

9. Record the **App client secret**. We will need this secret to generate an ID Token that will be made temporarily available for 60 minutes by default. You can customize this later if needed.

![Section5 Output WAF](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-get_app_client_secret.png)

</details>

Take note of **App client secret**. Other required values such as **user pool ID** and **App client ID** are available in the **Output** section of the current cloudformation stack. Record these before moving to the next step.


### 5.3. Create Cognito user pools as Authorizer in API Gateway Console.

Amazon Cognito user pools are used to control who can invoke REST API methods. We now need to integrate the API with the Amazon Cognito user pool.

1. Go to **API Gateway** in the AWS console and select API called **wa-lab-rds-api**.

![Section5 API without Authorizer](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-api_without_authorizer.png)

2. From the main navigation pane, choose **Authorizers** and click **Create New Authorizer** button.
* Type an authorizer name in Name.
* Select **Cognito** as Authorizer Type.
* Select your Cognito user pool (this should start with WAUserPool).
* For Token source, type **Authorization** as the header name to pass the identity or access token.

Your configuration should be similar to the screenshot below:


![Section5 Create New Authorizer](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-create_new_authorizer.png)

* When you have completed the configuration, click on **Create**.

3. You have now successfully added an **Authorizer**:


![Section5 Output WAF](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-authorizer.png)

4. From the main navigation pane, choose **Resource** to configure a COGNITO_USER_POOLS authorizer on methods.


**Before you click Method Request, ensure that you refresh your browser**. If you do not do this, our new **Authorizer** will not appear when trying to associate with Method Request.


Select **Method Request** as shown:


![Section5 Method Request](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-method_request.png)

5. Choose the pencil icon next to **Authorization**. You should be able to see the **Authorizer** we created in the drop-down list.
* To save the settings, choose the check mark icon

![Section5 Select Authorizer](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-select_new_authorizer.png)

6. Since we made a change, we need to re-deploy the API. Complete the following steps:

* Choose **Action**.
* **Deploy API**.

![Section5 Deploy API](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-deploy_API.png)

7. Select Development stage as **Dev** from the drop-down list. Click Deploy.

![Section5 Select Stage](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-select_stage.png)

8. Your API is now using COGNITO_USER_POOLS as Authorizer. Only users registered in COGNITO_USER_POOLS can access your API.

![Section5 Authorizer Association](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-authorizer_association.png)

### 5.4. Add Authorization header in CloudFront.
By default, CloudFront doesn't consider headers when caching your objects in edge locations. Now your request must have an **Authorization** header with a valid ID Token to access API. Therefore, we need to configure CloudFront to forward headers to the API Gateway. Further details on this can be found [here](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/header-caching.html)

1. Go to **CloudFront** in the AWS console and select your CloudFront distribution.
* From within the distribution, select the **Behaviors** tab.
* Select Cache Behavior associated with our API Gateway using the tick box.
* Click **Edit**.


![Section5 Forward Header](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-forward_header.png)

2. Select **Authorization** from the list of available headers and choose **Add**. To forward a custom header, enter the name of the header in the field, and choose Add Custom. Click **Yes,Edit**.

![Section5 Whitelist Headers](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-whitelist_header.png)


If you test out without Authorization header now, you will still see data being returned as it's served from CloudFront edge caches. Let's invalidate files to prevent this from happening.


3. Invalidate files from CloudFront edge.
* Select **Invalidation** tab.
* Select **Create Invalidation**
* Type /* , then click Invalidate button.

![Section5 Whitelist Headers](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-invalidate_files.png)


### 5.5. Generate an ID Token and send a request with ID Token.
After successful completing authentication, Amazon Cognito returns user pool tokens to your app. You can use these tokens to grant your users access to the API Gateway. Amazon Cognito user pools implements ID, access, and refresh tokens as defined by the OpenID Connect (OIDC) open standard. 

In this lab, we will use an ID Token that is a JSON Web Token (JWT) that contains claims about the identity of the authenticated user such as name, email, and phone_number.

1. In **Cloud9**, we will test with both CloudFrontEndpoint and APIGatewayURL to see the difference.

* Execute the script called sendRequest.py with the argument of your **CloudFrontEndpoint**.
```
python sendRequest.py 'CloudFrontEndpoint'
```
You will get **"Unauthorized"** response with **401 response code**.


![Section5 Test API Cloud9](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-test_api_cloud9.png)


You must send Authorization header with valid ID token to access API now.


2. Let's generate an ID token with your username and password, Cognito user pool ID, App client ID, and App secret.
We took note of **App client secret** when we signed up with Cognito.
Other required values such as **user pool ID, App client ID** are available in **Output** section of the current cloudformation stack
```
 python getIDtoken.py <username> <user_password> <user_pool_id> <app_client_id> <app_client_secret>
```
You should be able to generate ID Token successfully. This will be valid for 60 minutes.

![Section5 Generate cognito ID Token](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-generate_cognito_id_token.png)

3. Copy your **ID Token** you generate above. Send a request with this valid ID Token.
```
python sendRequest.py 'CloudFrontEndpoint' ID_Token
```
You should be seeing your data as expected with a **200 response code**. Now you must have provided an ID Token as the Authorization header to access your API only through CloudFront.


![Section5 Send Request With Cognito ID Token](https://github.com/sssalim-aws/Multilayered_API_Security_with_Cognito_and_WAF/blob/main/Images/section5/section5-send_request_with_cognito_id_token.png)


Ensure that you complete the tear down instructions in the final section to remove resources created in this lab.


### Teardown

Follow below steps to remove all resources related to this Lab.
Move to next steps only after the resource is deleted successfully.

1. Delete the S3 bucket referred in Step 3 of [Section 1.1](#11-deploy-cloudformation-template).

2. Delete Lambda function referred in Step 5 of [Section 2.2](#22-configure-password-rotation).

3. Delete CloudFormation the `walab-cdn-waf-cognito` stack refered in Step 1-3 of [Section 3.1](#31-deploy-security-enhancement).

4. Delete CloudFormation the `walab-api` stack refered in Step 7 of [Section 1](#step-1-deploy-the-lab-base-infrastructure).

5. Delete CloudFormation stack starting with `
SecretsManagerRDSMySQLRotationSingleUser` (This is stack was created as part of Step 2 in [Section 2.2](#22-configure-password-rotation))



### Survey

Let us know what you thought of this lab and how we can improve the experience for you in the future by completing [this session poll](https://amazonmr.au1.qualtrics.com/jfe/form/SV_ehwTCMiRy46skbY?Session=HOL006). Participants who complete the surveys from AWS Innovate - Modern Applications Edition will receive a gift code for USD25 in AWS credits1, 2 & 3. AWS credits will be sent via email by November 30, 2022. Note: Only registrants of AWS Innovate - Modern Applications Edition who complete the surveys will receive a gift code for USD25 in AWS credits via email.
1. AWS Promotional Credits Terms and conditions apply: https://aws.amazon.com/awscredits/
2. Limited to 1 x USD25 AWS credits per participant.
3. Participants will be required to provide their business email addresses to receive the gift code for AWS credits.