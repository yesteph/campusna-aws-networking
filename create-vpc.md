# Create a VPC and related resources (45 minutes!)

In this exercise, you will create a VPC, then access an EC2 instance you would have launched. This instance is a web application (PhpMyAdmin) which will connect to a Relational Database Management System (RDBMS) running in another VPC.

To get tha pplication working, you will need to create VPC peering and configure correctly the route tables.

To access the EC2 instance, your VPC must get the following resources:
* Internet Gateway
* a public subnet
* a private subnet
* route tables with correct rules

Furthermore, security groups must be created by you.

[Schema.png](./schema.png)


NOTE: the setup presented here is just for a hands-on. In real life, we would setup High Availability using several availability zones for all the components. You will see that in a later lesson.

## Pre-requisites

Each participant must have a dedicated AWS account and access to the WebConsole with Administrator Access.

Inside an account, the Terraform configuration in `./terraform` must have been applied to create a VPC, subnets, route tables and a RDS instance in region `eu-west-3`.

## Create a VPC 

Connect to the [AWS WebConsole](https://aws.amazon.com/console/).
Ensure you are on the Paris region. If not, select it. 

[screenshot](./res/regions.png)

Then go to the VPC service and click on the `You VPCs` link and `Create VPC` button.
Enter the following attributes:
* Name it: `my-vpc1`
* IPV4 CIDR: `10.0.0.0/16`
* No need to IPV6 CIDR.

Keep the default values and click on `Create`.

You should see your VPC created quickly.

## Create subnets

You will split you VPC in two subnets: one public and on private.

NOTE: in the real world to ease High Availability, we would deploy 4 subnets to ensure each type of subnet is on at least 2 availability zones.

In our setup, the public subnet will allow instances to access Internet and be accessed from Internet. About the private subnets, instances inside will not access to Internet. That mean we will not deploy Nat gateways.

### Create the public subnet

In the left navigation pane, click on the `Subnets` link and `Create subnet` button.

Enter the following attributes:
* VPC id: select the VPC `my-vpc1`
* Availability zone: `eu-west-3a`
* Name it: `my-public-subnet1`
* CIDR: `10.0.0.0/24`

Keep the default values and click on `Create`.

You should see your subnet created quickly.

Right now, our subnet is just a network space without any specificity about its visibility to/from Internet. 
But we want a **public** subnet. This supposes a route table targetting an Internet Gateway (see it later) and a creation by default of Public IP for any resource inside this subnet.

Select the subnet `my-public-subnet1` then click on `Actions` / `Edit subnet actions`. Then enable `Auto assignment public IPv4 address`.


### Create the private subnet

For the private subnet, in the left navigation pane, click on the `Subnets` link and `Create subnet` button.

Enter the following attributes:
* VPC id: select the VPC `my-vpc1`
* Availability zone: `eu-west-3a`
* Name it: `my-private-subnet1`
* CIDR: `10.0.1.0/24`

Keep the default values and click on `Create`.

You should see your subnet created quickly.

## Make 'public' the public subnet

### Create an Internet Gateway

You must create an Internet Gateway to declare it the route table of the public subnet.

In the left navigation pane, click on the `Internet Gateway` link and `Create internet gateway` button.

Name it `my-internet-gateway1` and click `Create`.


Now we must associate this gateway to our VPC: once created, click on `Actions`/ `Attach to VPC` and select the VPC `my-vpc1`.

Click `Attach the Internet Gateway`.

### Configure your route tables

Each subnet is associated to a route table.

A VPC has a default route table. Currently, this route table is used by the **public** and **private** subnets.

Here, the public subnet must target the internet gateway for the default destination - 0.0.0.0/0. For that, you must create a new route table associtated to the **public** subnet.

Create a new route table for the public subnet. In the left navigation pane, click on the `Route tables` link and `Create route table` button.

Name it `my-public-route-table1` and select the VPC `my-vpc1`.

Now, add routes by selecting the newly created route table and clicking on `Edit route tables`. Add the public route: for the destination `0.0.0.0/0` set the target **Internet gateway** to `my-internet-gateway1`. 

Then `Save Changes`.

Now, you need to indicate the public subnet must use this route table. On the details screen of the route table `my-public-route-table1`, edit the `Subnet associations` to add the subnet `my-public-subnet1`.

## Create an EC2 instance

### Security group

A security group is a firewall applied to each EC2 instance or Elastic Network Interface. Each EC2 instance can have its own security group.

You must create a security group to allow HTTP connection to the EC2 instance you will create just after.

In the left navigation pane, click on the `Security / Security Groups` link and `Create security group` button.

Enter the following attributes:
* Name: `my-web-server1`
* Description: `Allow HTTP access from anywhere`
* VPC id: select the VPC `my-vpc1`

Add an inbound rule:
* Type: `HTTP``
* Source: `Anywhere IPV4`

Keep the default outbound rule which allow all traffic leaving the instance.
Click on the `Create Security Group` button.

### EC2 Instance

Now, you will create a PHP web server running on an EC2 instance.

Go to the `EC2` service then in the left pane, click on `Instances` and `Launch Instance` button.

Enter the following attributes:
* AMI: `Amazon Linux 2`
* Instance type: `t3.micro`
* On the `Configure Instance Details` screen:
  * Network: choose VPC `my-vpc1`
  * Subnet: `my-public-subnet1`
  * Select `ssm` for the IAM role.
  * Scroll down the web page to see the `Advanced Details/ User data` section and copy this content:
```sh
#!/bin/bash

# Install APACHE, PHP, mariadb, PHPMyAdmin
sudo yum update -y
sudo amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2  
sudo yum install -y httpd mysql  jq php-mbstring php-xml
wget https://www.phpmyadmin.net/downloads/phpMyAdmin-latest-all-languages.tar.gz
mkdir /var/www/html/phpMyAdmin && tar -xvzf phpMyAdmin-latest-all-languages.tar.gz -C /var/www/html/phpMyAdmin --strip-components 1
rm -rf phpMyAdmin-latest-all-languages.tar.gz

# Start Apache
sudo systemctl start httpd
sudo systemctl enable httpd

# Retrieve the configuration file for phpMyAdmin. In particular the address of the RDS instance
aws secretsmanager get-secret-value --secret-id phpmyadmin_config  --region eu-west-3|jq -r '.SecretString'  > /var/www/html/phpMyAdmin/config.inc.php

```
* Click `Add Storage`- no storage modification
* Click `Add Tags`
  * add a key/value: `Name`/`my-web-server1`
* Configure Security groups
  * Select an existing security group
  * Choose `my-web-server1` for the securty group

Click on `Review and launch` then `Launch`. You will get a message asking to select a key pair, choose the existing key pair `student` if it is not done yet.

Then launch the instance.

Click on `View instances`. You should see the created EC2 instance.

### Connect to the EC2 instance

Select the instance and copy its public IP.

Open it in a web browser with the **`htttp://`** protocol, not **`https://`**.

You should get an Apache web page.

You should be able to access the phpMyAdmin which is running on the `/phpMyAdmin` url.
But if you try to login to the MySQL database with login: `applicationuser` and password: `a_secured_password` you will get an error as the database is not reachable. Indeed, it is in another VPC ; you must interconnect the two VPCs.

## VPC peering

### Create the VPC peering

A VPC named `db-vpc` is created on CIDR 10.1.0.0/8.
This VPC contains a running database.

Go to the `VPC` service.
Then click in the left pane onto the `Peering connections` link and `Create Peering connection` button.

* Name: `my-peering-1`
* VPC ID (requester): `my-vpc1`
* VPC ID (accepter): `my-db-vpc1`

Click on the `Create peering connection button`.

The owner of the target must accept a VPC peering connection. Fortunately, you are the owner of the both VPCs so you can accept the request!
On the details screen of the created peering connection, click on the `Actions` button then `Accept the peering connection`.

### Use the peering in your route tables

You must update route tables in both VPCs to target the peering.

Select the route table `my-public-route-table-1` and modify its routes to add a new route:
* Destination: `10.1.0.0/16` (range of the `my-db-vpc1` vpc)
* Target: select `Peering Connection` then `my-peering-1`

Now, select the route table `my-db-vpc1-private` which is used by the subnets of the database and modify its routes to add a new route:
* Destination: `10.0.0.0/16` (range of the `my-vpc1` vpc)
* Target: select `Peering Connection` then `my-peering-1`

### Test the cross VPC application

Now if you open the phpMyAdmin application again and try to connect to the RDS instance, you should see the list of databases available on the RDS instance:
* URL: `http://PUBLIC_IP_OF_YOUR_EC2/phpMyAdmin`
* login: `applicationuser`
* password: `a_secured_password`


Congratulations! You have finished this hands-on.