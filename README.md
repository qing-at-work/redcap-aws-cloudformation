# Deploy a Project REDCap environment on AWS using automation and architectural best practices
This repository contains AWS CloudFormation templates to automatically deploy a Project REDCap environment that adheres to AWS architectural best practices.  In order to use this automation you must supply your own copy of the Project REDCap source files.  These are available for qualified entities at https://www.project-redcap.org/.  Once you have downloaded your source files then you can follow the below instructions for deployment.

In their own words: "REDCap is a secure web application for building and managing online surveys and databases. While REDCap can be used to collect virtually any type of data (including 21 CFR Part 11, FISMA, and HIPAA-compliant environments), it is specifically geared to support online or offline data capture for research studies and operations. The REDCap Consortium, a vast support network of collaborators, is composed of thousands of active institutional partners in over one hundred countries who utilize and support REDCap in various ways."

## Project REDCAp on AWS architecture and features
The features of using this architecture are as follows:
* A complete and ready-to-use Project REDCap environment is automatically deployed in about 20 minutes.
* Project REDCap is deployed in an isolated, three-tiered Virtual Private Cloud
* The environment enables automatic scaling up and down based on load
* Data is encrypted by default at rest and in flight (in accordance with HIPAA)
* Managed services are used that provide automated patching and maintanence of OS, middleware, and database software
* Database backups are performed automatically to enable operational and disaster recovery
* [The design results in a reasonable monthly cost](https://calculator.s3.amazonaws.com/index.html#r=IAD&key=calc-42CFC1C0-3356-4A35-8697-0A9567A8EA3B) 

A high-level diagram showing how the different functions of Project REDCap map to AWS Services is shown below.  
![alt-text](https://github.com/JamesSWiggins/project-redcap-aws-automation/raw/master/images/AWS%20Project%20REDCap%20Block%20Diagram.png "AWS Project REDCap High-Level Diagram")

Starting from the user, public Internet DNS services are (optionally) provided by **Amazon Route 53**.  This gives you the ability to automatically add a domain to an existing hosted zone in Route 53 (i.e. redcap.example.edu if example.edu is already hosted in Route 53).  In addition, if you are deploying a new domain to Route 53, an SSL certificate can be automatically generated and applied using **AWS Certificate Manager (ACM)**.  This enables HTTPS communication and ensures the data sent from the users is encrypted in-transit (in accordance with HIPAA).  HTTPS communication is also used between the Application Load Balancers and the Project REDCap servers.

**AWS Elastic Beanstalk** is used to deploy the Project REDCap application onto Apache/PHP Linux servers.  Elastic Beanstalk is an easy-to-use service for deploying and scaling web applications. It covers everything from capacity provisioning, load balancing, regular OS and middleware updates, autoscaling, and high availability, to application health monitoring. Using a feature of Elastic Beanstalk called ebextensions, the Project REDCap servers are customized to use an encrypted storage volume for the middleware application logs.

**Amazon Relational Database Service (RDS)** with Amazon Aurora MySQL is used to provide an (optionally) highly available database for Project REDCap.  Amazon Aurora is a relational database built for the cloud that combines the performance and availability of high-end commercial databases with the simplicity and cost-effectiveness of open-source databases. It provides cost-efficient and resizable capacity while automating time-consuming administration tasks such as hardware provisioning, database setup, patching, and backups. It is configured for high availability and uses encryption at rest for the database and backups, and encryption in flight for the JDBC connections.  The data stored inside this database is also encrypted at rest.

**Amazon Simple Storage Service (S3)** is used as a file repository for files uploaded through Project REDCap.  S3 is designed to deliver 99.999999999% durability, and stores data for millions of applications used by market leaders in every industry. S3 provides comprehensive security and compliance capabilities that meet even the most stringent regulatory requirements.  The S3 bucket used to store these files is encrypted using AES-256.  Please note that the present integration from Project REDCap necessitates that all files be stored in the Northern Virginia AWS Region (us-east-1).

**Amazon Simple Email Service (SES)** is used enable Project REDCap to send emails to users.  SES is a powerful, affordable, and highly-scalable email sending and receiving platform for businesses and developers that integrates seamlessly with applications and with other AWS products.  SES provides a reliable SMTP gateway without the need to maintain a separate SMTP server.

A more detailed, network oriented diagram of this environment is shown following.
![alt-text](https://github.com/JamesSWiggins/project-redcap-aws-automation/blob/master/images/AWS%20Project%20REDCap%20Network%20Diagram.png "AWS Project REDCap Network Diagram")

## Project REDCap on AWS deployment instructions
Before deploying an application on AWS that transmits, processes, or stores protected health information (PHI) or personally identifiable information (PII), address your organization's compliance concerns. Make sure that you have worked with your internal compliance and legal team to ensure compliance with the laws and regulations that govern your organization. To understand how you can use AWS services as a part of your overall compliance program, see the [AWS HIPAA Compliance whitepaper](https://d0.awsstatic.com/whitepapers/compliance/AWS_HIPAA_Compliance_Whitepaper.pdf). With that said, we paid careful attention to the HIPAA control set during the design of this solution.

### Pre-requisite tasks
0.1. Follow the instructions on the [Project REDCap website](https://www.project-redcap.org/) to obtain a copy of the Project REDCap source files.

0.2. [Create a private S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html) and [upload your Project REDCap source file](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/upload-objects.html) into it.  Ensure that you do not make either the bucket or the source file publicly readable.

0.3. [Obtain your Amazon SES SMTP Credentials](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/smtp-credentials.html) using the Amazon SES console.  Download your credentials and store them in a safe place.

0.4. AWS has strict safeguards in place regaridng email to prevent inappropriate use.  In order to send outbound email you must [verify the specific email address(es) from which you will be sending mail](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-email-addresses-procedure.html) and/or [verify the domain from which you will be sending mail](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-domain-procedure.html).  In addition, if you intend to send email to an email address or domain other than those you have verified, you must [submit a request to be moved out of the Amazon SES sandbox](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/request-production-access.html).  If these steps are not taken then sending e-mail from your Project REDCap application will not work properly.

It is also strongly recommended that you implement a [method to handle email bounces and complaints that might occur](https://aws.amazon.com/blogs/messaging-and-targeting/handling-bounces-and-complaints/).

#### If you intend to use Route 53 for DNS or ACM to provide an SSL certificate
0.5. Automatically provisioning and applying an SSL certificate with this CloudFormation tempalte using ACM requires the use of Route 53 for your DNS service.  If you have not already done so, [create a new Route 53 Hosted Zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html), [transfer registration of an existing domain to Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/domain-transfer-to-route-53.html), or [transfer just your DNS service to Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/MigratingDNS.html).

If you do not intend to use Route 53 and ACM to automatically generate and provide an SSL certificate, [an SSL certificate can be applied to your environment after it is deployed](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/configuring-https-elb.html).

### Deployment Instructions
0. This template must be run by an AWS IAM User who has sufficient permission to create the required resources.  These resources include:  VPC, IAM User and Roles, S3 Bucket, EC2 Instance, ALB, Elastic Beanstalk, Route53 entries, and ACM certificates.  If you are not an Administrator of the AWS account you are using, please check with them before running this template to ensure you have sufficient permission.  

1. From your AWS account, [open the CloudFormation Management Console](https://console.aws.amazon.com/cloudformation/) and choose **Create Stack**.  From there, copy and paste the following URL in the **Specify an Amazon S3 template URL**, and choose **Next**.  https://s3.amazonaws.com/project-redcap-aws-automation/00-master-rc.yaml
![alt-text](https://github.com/JamesSWiggins/project-redcap-aws-automation/blob/master/images/redcap_cfn_select_template.png "CFN Select Template")


2. On the next screen, provide a **Stack Name** (this can be anything you like, but no uppercase letters please) and a few other parameters for your Project REDCap environment.  A description is provided for each parameter to explain it's function.  Please keep in mind that the **Elastic Beanstalk Endpoint Name** must be unique within your AWS Region.  You can check to see if an endpoint name is in use by checking for an existing DNS entry using the 'nslookup' command (nslookup (EBEndpoint).(region).elasticbeanstalk.com).  If the command returns an IP address, that means that the name is in use and you should pick a different name.  When you've provided appropriate values for the **Parameters**, choose **Next**.

3. On the next screen, you can provide some other optional information like tags at your discretion, or just choose **Next**.

4. On the next screen, you can review what will be deployed. At the bottom of the screen, there is a check box for you to acknowledge that **AWS CloudFormation might create IAM resources with custom names**. This is correct; the template being deployed creates four custom roles that give permission for the AWS services involved to communicate with each other. Details of these permissions are inside the CloudFormation template referenced in the URL given in the first step. Check the box acknowledging this and choose **Next**.

5. You can watch as CloudFormation builds out your Project REDCap environment. A CloudFormation deployment is called a *stack*. The parent stack creates several child stacks depending on the parameters you provided.  When all the stacks have reached the green CREATE_COMPLETE status, as shown in the screenshot following, then the Project REDCAp architecture has been deployed.  Select the **Outputs** tab to find your Project REDCap environment URL.
![alt-text](https://github.com/JamesSWiggins/project-redcap-aws-automation/blob/master/images/redcap_cfn_stack_complete.png "CFN Stack Complete")

6. After clicking on the provided URL, you will be taken to the Project REDCap login screen.  You can login by using the username 'redcap_admin' and the password you provided in the **DB Master Password Parameter**.  You will immediately be asked to change the password.

### Congratulations
At this point, you have a fully functioning and robust Project REDCap environment to begin using.
