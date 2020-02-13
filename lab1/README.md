# LAB 1 - Creating Redshift Clusters
이 실습에서는 새로운 Redshift Cluster를 시작하고 연결을 설정하고 JDBC 클라이언트 도구를 구성합니다.

## Contents
* [시작하기 앞서](#before-you-begin)
* [필요 사항](#prerequisites)
* [Cloud Formation](#cloud-formation)
* [보안 설정](#configure-security)
* [Redshift Cluster 기동](#launch-redshift-cluster)
* [Client 도구 설정](#configure-client-tool)
* [Sample Query를 돌려봅시다](#run-sample-query)
* [떠나기 전 정리](#before-you-leave)

## 시작하기 앞서
* Determine and capture the following information and login to the [AWS Console](https://console.aws.amazon.com/). If you are new to AWS, you can [create an account](https://portal.aws.amazon.com/billing/signup).
  * [Your-AWS_Account_Id]
  * [Your_AWS_User_Name]
  * [Your_AWS_Password]
* Determine the [AWS Region Name] and [AWS Region Id] which is closest to you and switch your console to that [Region](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html).  

## Prerequisites
* While Amazon Redshift does provide a web-based [Query editor](https://console.aws.amazon.com/redshift/home?#query:) for executing simple queries which complete in under 3 minutes, for these labs, it is recommended you install a third-party tool.  We will use [SQL Workbench/J](http://www.sql-workbench.net).
* Once you have installed your third-party tool, you will need either a JDBC or ODBC driver.  Amazon Redshift offers JDBC and ODBC drivers for download. See [Connecting to an Amazon Redshift Cluster Using SQL Client Tools](https://docs.aws.amazon.com/redshift/latest/mgmt/connecting-to-cluster.html) from the Amazon Redshift Documentation website.

## Cloud Formation
To launch this cluster and configure security automatically using cloud formation, use the following link and skip ahead to [Configure Client Tool](#configure-client-tool).  
[![Launch](../images/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=ImmersionLab1&templateURL=https://s3-us-west-2.amazonaws.com/redshift-immersionday-labs/lab1.yaml)

## Configure Security
### VPC
Create or identify a VPC where you will launch your Redshift cluster.  For our purposes we will create a **VPC** to isolate the traffic.
```
https://console.aws.amazon.com/vpc/home?#CreateVpc:
```
![](../images/VPC.png)
### InternetGateway
Create or identify an Internet Gateway.  For our purposes we will create an **Internet Gateway**. Once created, select the Internet Gateway and attach it to the VPC created earlier.  
```
https://console.aws.amazon.com/vpc/home?#Create%20Internet%20Gateway:
```
![](../images/InternetGateway.png)
![](../images/InternetGatewayAttach1.png)
![](../images/InternetGatewayAttach2.png)
### Subnets
Create or identify a subnet with a default route to an Internet Gateway.  For our purposes we will create a **Subnet** attached to the previously created VPC in two different Availability Zones to improve fault tolerance.
```
https://console.aws.amazon.com/vpc/home?#CreateSubnet:
```
![](../images/Subnet1.png)
![](../images/Subnet2.png)
### Route Table
If you have created new subnets, you must create a **Route Table** with the default route pointed to the internet gateway and with the new subnets added.
```
https://console.aws.amazon.com/vpc/home?#CreateRouteTable:
```
![](../images/Route.png)
![](../images/EditRoute.png)
![](../images/EditSubnet.png)
### Subnet Group
Create a Redshift **Cluster Subnet Group** containing the two subnets you created earlier by clicking the add all subnets for this VPC button.
```
https://console.aws.amazon.com/redshiftv2/home?#subnet-groups
```
<table><tr><td><img src=../images/SubnetGroup.png></td></tr></table> 

### Security Group
Create a **Security Group** associated to the VPC you created earlier.  Edit the Security Group to create a rule which allows incoming connections from your IP Address.
```
https://console.aws.amazon.com/vpc/home#SecurityGroups:sort=groupId
```
![](../images/SecurityGroup.png)
### S3 Access
Create an **IAM Role** with the type "Redshift" and the use-case of "Redshift - Customizable" and attach the AmazonS3ReadOnlyAccess and AWSGlueConsoleFullAccess policies to the role.
```
https://console.aws.amazon.com/iam/home?#/roles$new?step=type
```
![](../images/Role.png)

## Launch Redshift Cluster
Navigate to the **Amazon Redshift Dashboard** and click on the "Create Cluster" button.  
```
https://console.aws.amazon.com/redshiftv2/home?#clusters
```
* Cluster Configuration - Choose the node type and set the number of nodes.  For these labs a dc2.large node type with 4 nodes will be suitable. 
<table><tr><td><img src=../images/CreateCluster1.png></td></tr></table> 

* Cluster Details - Enter values as appropriate for your organization.  Note the Master user password as you will not be able to retrieve this value later.
<table><tr><td><img src=../images/CreateCluster2.png></td></tr></table> 

* Cluster Permissions - Select the Role which you identified or created earlier to associate to the cluster, and click **Add IAM role**
<table><tr><td><img src=../images/CreateCluster3.png></td></tr></table> 

* Additional Configuration - Disable **Use defaults** and choose the VPC, Subnet Group, and VPC Security group you identified or created earlier.
<table><tr><td><img src=../images/CreateCluster4.png></td></tr></table> 

Leave the remaining settings with their default values.  Click **Create Cluster** to launch the Redshift cluster.

## Configure Client Tool
* See [Prerequisites](#prerequisites) for more details on downloading and installing [SQL Workbench/J](http://www.sql-workbench.net) and the [Redshift JDBC Driver](https://docs.aws.amazon.com/redshift/latest/mgmt/connecting-to-cluster.html). 
* Launch SQL Workbench/J and navigate to [File | Manage Drivers].
* Select "Amazon Redshift" and set the driver Library location to where you downloaded the Redshift JDBC Driver. Click Ok.
![](../images/Library.png)
* Navigate to [File | Connect Window] to create a new connection profile and modify the following settings and once complete click on the "Test Connection" button.
  * Name - "LabConnection"
  * Driver - Amazon Redshift (com.amazon.redshift.jdbc.Driver)
  * URL - Find this by navigating to the [Cluster List](https://console.aws.amazon.com/redshiftv2/home?#clusters), selecting your cluster, clicking on **Properties** and copying the **Endpoint** located in the **Connection details**.  
  ![](../images/JDBCUrl.png)
  * Username - [Master user name]
  * Password - [Master user password]
  * Autocommit - Enabled
  
![](../images/Connection.png)

## Run Sample Query
* Run the following query to list the users within the redshift cluster.  
```
select * from pg_user
```
* If you receive the following results, you have established connectivity and this lab is complete.  
![](../images/Users.png)

## Before You Leave
If you are done using your cluster, please think about decommissioning it to avoid having to pay for unused resources.
