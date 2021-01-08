# Migrating a commercial database to open source with AWS SCT and AWS DMS 

[![Build Status](https://github.com/aws-samples/aws-dms-sql-server/workflows/Publish%20Version/badge.svg)](https://github.com/aws-samples/aws-dms-getting-started-blog/actions)

### Index

* [Introduction](#introduction)
* [Architecture](#architecture)
* [Environment Configuration](#environment-configuration)
  * [Generate Key Pair](#generate-key-pair)
  * [Launch CloudFormation Stack](#launch-cloudformation-stack)
  * [Summary](#summary)
* [Clean up](#clean-up)
* [Security](#security)
* [License](#license)

### Introduction

This repository demonstrates the ease of database migration from commercial database engines such as Microsoft SQL Server and Oracle to open source engines such as Amazon Aurora (MySQL) and Amazon Aurora (PostgreSQL).

This section describes the steps to provision the AWS resources that are required for this database migration walkthrough. We use [AWS CloudFormation][cfn] to create a network topology that includes a simple [Amazon Virtual Private Cloud (Amazon VPC)][vpc] with 3 public subnets to deploy the [AWS Database Migration Service (AWS DMS)][aws-dms] replicaiton instnaces, as well as [Amazon Relational Database Service (Amazon RDS)][rds] instance for the target database. Additionally, it provisions an [Amazon Elastic Compute Cloud (EC2)][ec2] instance to host the tools that we use in this migration like the [AWS Schema Conversion Tool (AWS SCT)][aws-sct]. AWS CloudFormation simplifies provisioning the infrastructure, so we can concentrate on tasks related to data migration.

After you have completed the migration, you can refer to the [Environment Cleanup][env-cleanup] guide to delete the resources in your account to stop incurring additional costs. 

### Architecture

When you create a stack from the AWS CloudFormation template, it provisions the following resources:
- A VPC with default CIDR (10.20.0.0/16) with three public subnets in your region. 
    - Subnet1 default address 10.20.1.0/24 in Availability Zone (AZ) 1
    - Subnet2 default address 10.20.2.0/24 in Availability Zone (AZ) 2
    - Subnet3 default address 10.20.3.0/24 in Availability Zone (AZ) 3
    
- A DB subnet group that includes Subnet1, Subnet2, and Subnet3.

- An Amazon EC2 instance to host the source databae for Microsoft SQL Server migraiton, AWS Schema Conversion Tool (AWS SCT), and other tools such as SQL Server Developer, and pgAdmin.
  - m5.large or equivalent instance class
  - A Security Group with port 3389 (RDP) open to 0.0.0.0/0 (access from anywhere)  
  
- An [Amazon Aurora (MySQL)][aurora] database instance as the target database for the Microsoft SQL Server migration section.
  - No replicas
  - db.r4.large or equivalent instance class
  - Port 3306
  - Default option and parameter groups
  
- An Amazon RDS Oracle Enterprise Edition (EE) as the source database for the Oracle mgiration section. 
  - Single-AZ setup
  - db.r4.large or equivalent instance class
  - Port 1521
  - Default option and parameter groups
  
- An Amazon Aurora (PostgreSQL) DB instance as the target database for the Oracle migration section.
  - No replicas
  - db.r4.large or equivalent instance class
  - Port 1433
  - Default option and parameter groups

The CloudFormation template gives you the flexibility to modify the VPC CIDR, and Amazon EC2 and Amazon RDS instance types using the input parameters as you launch the stack. 

We will use the AWS Management Console to provision the AWS DMS resources, such as the replication instance, endpoints, and tasks. 

## Environment Configuration
1. Login to [AWS Management Console][console] using your credentials. 

2.	Click on the drop-down menu on the top right corner of the screen, and select one of the 10 supported regions for this tutorial:

| **Region Name** | **Region** |
| ------ | ------ |
| US East (N. Virginia) | us-east-1 |
| US East (Ohio) | us-east-2 |
| US West (Oregon) | us-west-2 |
| EU (Frankfurt) | eu-central-1 |
| EU (Ireland) | eu-west-1 |
| EU (London) | eu-west-2 |
| EU (Paris) | eu-west-3 |
| Asia Pacific (Tokyo) | ap-northeast-1 |
| Asia Pacific (Seoul) | ap-northeast-2 |
| Asia Pacific (Singapore) | ap-southeast-1 |
| Asia Pacific (Sydney) | ap-southeast-2 |

![\[AWS Management Console Region Selection\]](img/EnvConfig/EnvConfig01.png)

#### Generate Key Pair
In this step, you will generate an EC2 key pair that you will use to connect to the EC2 instance.

3. Click [here][key-pair] to navigate to the **Key Pair** section in the EC2 console. Ensure you are in the same region as you chose in the previous step. Then, click on the **Create Key Pair** button

![\[create-keypair1\]](img/EnvConfig/EnvConfig02.png)

4. Name the key pair **DMSKeyPair**, and then click **Create**.  At this point, your browser will download a file named **DMSKeyPair.pem**.  Save this file.  You will need it to complete the tutorial.

![\[create-keypair2\]](img/EnvConfig/EnvConfig03.png)

#### Launch CloudFormation Stack
In this step, you will use a CloudFormation (CFN) template to deploy the infrastructure for this database migration. You can download the CFN template [here][cfn-template].

5. Open the CloudFormation console, and click on **Create Stack** in the left-hand corner.

![\[create-stack\]](img/EnvConfig/EnvConfig04.png)

6. Select Template is ready, and choose Upload a template file as the source template. Then, click on Choose file and upload the [DatabaseMigration_CFN_Template.yaml][cfn-template]. Click Next.

![\[add-template\]](img/EnvConfig/EnvConfig05.png)

7. Populate the form as with the values specified below, and then click **Next**.

| **Input Parameter** | **Values** |
| ------ | ------ |
| **Stack Name** | A unique identifier without spaces. |
| **MigrationType** | Database that you want to migrate (Oracle or SQL Server). |
| **KeyName** | The KeyPair (DMSKeypair) that you created in the previous step. |
| **EC2ServerInstanceType** | An Amazon EC2 Instance type from the drop-down menu. Recommend using the default value. |
| **RDSInstanceType** | An Amazon RDS Instance type from the drop-down menu. |
| **VpcCIDR** | The VPC CIDR range in the form x.x.x.x/16. Defaults to 10.20.0.0/16 |
| **Subnet1CIDR** | The Subnet CIDR range for subnet 1 in the form x.x.x.x/24. Defaults to 10.20.1.0/24 |
| **Subnet2CIDR** | The Subnet CIDR range for subnet 2 in the form x.x.x.x/24. Defaults to 10.20.2.0/24 |
| **Subnet3CIDR** | The Subnet CIDR range for subnet 3 in the form x.x.x.x/24. Defaults to 10.20.3.0/24 |
| **YourIPAddress** | Enter your IP address in the form x.x.x.x/32 You can find your IP on this website http://ipinfo.io/ip . Defaults to 0.0.0.0/0|

*Note: The resources that are created here will be prefixed with whatever value you specify in the Stack Name.  Please specify a value that is unique to your account.*

![\[input-parameters\]](img/EnvConfig/EnvConfig06.png)

8. On the **Stack Options** page, accept all of the defaults and click **Next**.

9. On the **Review** page, scroll down to the bottom and check "I acknowledge that AWS CloudFormation might create IAM resources." Then, click **Create stack**.

![\[review-stack\]](img/EnvConfig/EnvConfig07.png)

10.	At this point, you will be directed back to the CloudFormation console and will see a status of **CREATE_IN_PROGRESS**.  Please wait here until the status changes to **COMPLETE**.

![\[stack-progress\]](img/EnvConfig/EnvConfig08.png)

11.	Once CloudFormation status changes to **CREATE_COMPLETE**, go to the **Outputs** section.

12.	Make a note of the **Output** values from the CloudFormation environment that you launched as you will need them for the remainder of the tutorial:
    - Microsoft SQL Server to Amazon Aurora (MySQL) migration output
    
    ![\[sqlserver-outputs\]](img/EnvConfig/EnvConfig09.png)
    
    - Oracle to Amazon Aurora (PostgreSQL) migration output
    
    ![\[oracle-outputs\]](img/EnvConfig/EnvConfig10.png)
    
#### Summary

You have completed the configuration of your environment. Please make sure that you take note of the output values and location of the DMSKeyPair.pem file generated during this setup.  You will use these values later in the migraiton process. Now you can select which migration walkthrough you want to follow:

- [Microsoft SQL Server to Amazon Aurora (MySQL)](SQLServerMigration.md)
- [Oracle to Amazon Aurora (PostgreSQL)](OracleMigration.md)

## Clean up

After you are done migrating the sample database, use the [Environment Cleanup](EnvironmentCleanup.md) section to remove the resources you created during this walk through to avoid additional charges.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.


[console]: <https://console.aws.amazon.com/>
[ec2-console]: <http://amzn.to/2atGc3r>
[dms-console]: https://console.aws.amazon.com/dms/
[env-cleanup]: </EnvironmentCleanup.md>
[aws-sct]: <https://aws.amazon.com/dms/schema-conversion-tool/?nc=sn&loc=2>
[aws-dms]: <https://aws.amazon.com/dms/>
[aurora]: <https://aws.amazon.com/rds/aurora/>
[ec2]: <https://aws.amazon.com/ec2/>
[vpc]: <https://aws.amazon.com/vpc/>
[download-sct]: <https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Installing.html>
[cfn]: <https://aws.amazon.com/cloudformation/>
[rds]: <https://aws.amazon.com/rds/>
[cfn-template]: </DatabaseMigration_CFN_Template.yaml>
[key-pair]: <http://amzn.to/2kcoMQp>
