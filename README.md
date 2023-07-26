# AWS Three Tier Web Architecture

## Description:

This project is a hands-on walk through of a three-tier web architecture in AWS. We will be manually creating the necessary network, security, app, and database components and configurations in order to run this architecture in an available and scalable manner.

## Audience:

It is intended for those who have a technical role. The assumption is that you have at least some foundational aws knowledge around VPC, EC2, RDS, S3, ELB and the AWS Console.

## Pre-requisites:

1. An AWS account. If you don’t have an AWS account, follow the instructions [here](https://aws.amazon.com/console/) and
   click on “Create an AWS Account” button in the top right corner to create one.
1. IDE or text editor of your choice.

## Architecture Overview

![Architecture Diagram](https://github.com/aws-samples/aws-three-tier-web-architecture-workshop/blob/main/application-code/web-tier/src/assets/3TierArch.png)

In this architecture, a public-facing Application Load Balancer forwards client traffic to our web tier EC2 instances. The web tier is running Nginx webservers that are configured to serve a React.js website and redirects our API calls to the application tier’s internal facing load balancer. The internal facing load balancer then forwards that traffic to the application tier, which is written in Node.js. The application tier manipulates data in an Aurora MySQL multi-AZ database and returns it to our web tier. Load balancing, health checks and autoscaling groups are created at each layer to maintain the availability of this architecture.

## Setup - Part 0

### Download Code from Github

git clone <repo>

### S3 Bucket Creation

1. Navigate to the S3 service in the AWS console and create a new S3 bucket.
   </image>
2. Select the region you want and give unique name to the bucket

### IAM EC2 Instance Role Creation

1. Navigate to the IAM dashboard in the AWS console and create an EC2 role.
2. Select EC2 as the trusted entity.
   </image>
3. When adding permissions, include the following AWS managed policies. You can search for them and select them. These policies will allow our instances to download our code from S3 and use Systems Manager Session Manager to securely connect to our instances without SSH keys through the AWS console.

   - AmazonSSMManagedInstanceCore
   - AmazonS3ReadOnlyAccess

4. Give your role a name, and then click Create Role.

## Networking and Security - Part 1

Now we will be building out the VPC networking components as well as security groups that will add a layer of protection around our EC2 instances, Aurora databases, and Elastic Load Balancers.

- Create an isolated network with the following components:
  - VPC
  - Subnets
  - Route Tables
  - Internet Gateway
  - NAT gateway
  - Security Groups

### VPC and Subnets

#### VPC Creation

1.  Navigate to the VPC dashboard in the AWS console and navigate to <b>Your VPCs</b> on the left hand side.
2.  Make sure VPC only is selected, and fill out the VPC Settings with a Name tag and a CIDR range of your choice.
    </image>

    <b>NOTE</b>: Make sure you pay attention to the region you’re deploying all your resources in. You’ll want to stay consistent for this workshop.

    <b>NOTE</b>: Choose a CIDR range that will allow you to create at least 6 subnets.

#### Subnet Creation

1. Next, create your subnets by navigating to Subnets on the left side of the dashboard and clicking Create subnet.
2. We will need six subnets across two availability zones. That means that three subnets will be in one availability zone, and three subnets will be in another zone. Each subnet in one availability zone will correspond to one layer of our three tier architecture. Create each of the 6 subnets by specifying the VPC we created in part 1 and then choose a name, availability zone, and appropriate CIDR range for each of the subnets.

   <b>NOTE</b>: It may be helpful to have a naming convention that will help you remember what each subnet is for. For example in one AZ you might have the following: Public-Web-Subnet-AZ-1, Private-App-Subnet-AZ-1, Private-DB-Subnet-AZ-1.

   <b>NOTE</b>: Remember, your CIDR range for the subnets will be subsets of your VPC CIDR range.

Your final subnet setup should be similar to this. Verify that you have 3 subnets across 2 different availability zones.
</image>

### Internet Connectivity

#### Internet Gateway

1. In order to give the public subnets in our VPC internet access we will have to create and attach an Internet Gateway. Click on Internet Gateway on the Dashboard.
2. Create your internet gateway by simply giving it a name and clicking Create internet gateway.
3. After creating the internet gateway, attach it to your VPC. You have a couple options on how to do this, either with the creation success message or the Actions drop down.
   </image>
4. Then, select the correct VPC and click Attach internet gateway.

#### NAT Gateway

1. In order for our instances in the app layer private subnet to be able to access the internet they will need to go through a NAT Gateway. For high availability, you’ll deploy one NAT gateway in each of your public subnets. Navigate to NAT Gateways on the left side of the current dashboard and click Create NAT Gateway.
2. Fill in the Name, choose one of the public subnets you have created, and then allocate an Elastic IP. Click Create NAT gateway.
   </image>
3. Repeat step 1 and 2 for the other subnet.

### Routing Configuration

1. Navigate to Route Tables on the left side of the VPC dashboard and click Create route table First, let’s create one route table for the web layer public subnets and name it accordingly.
   </image>
2. After creating the route table, you'll automatically be taken to the details page. Scroll down and click on the Routes tab and Edit routes.
   </image>
3. Add a route that directs traffic from the VPC to the internet gateway. In other words, for all traffic destined for IPs outside the VPC CDIR range, add an entry that directs it to the internet gateway as a target. Save the changes.
   </image>
4. Edit the Explicit Subnet Associations of the route table by navigating to the route table details again. Select Subnet Associations and click Edit subnet associations.
   </image>
   Select the two web layer public subnets you created eariler and click Save associations.
   </image>
5. Now create 2 more route tables, one for each app layer private subnet in each availability zone. These route tables will route app layer traffic destined for outside the VPC to the NAT gateway in the respective availability zone, so add the appropriate routes for that.
   </image>
   Once the route tables are created and routes added, add the appropriate subnet associations for each of the app layer private subnets.
   </image>

### Security Groups

1. Security groups will tighten the rules around which traffic will be allowed to our Elastic Load Balancers and EC2 instances. Navigate to Security Groups on the left side of the VPC dashboard, under Security.
2. The first security group you’ll create is for the public, internet facing load balancer. After typing a name and description, add an inbound rule to allow HTTP type traffic for your IP.
   </image>
3. The second security group you’ll create is for the public instances in the web tier. After typing a name and description, add an inbound rule that allows HTTP type traffic from your internet facing load balancer security group you created in the previous step. This will allow traffic from your public facing load balancer to hit your instances. Then, add an additional rule that will allow HTTP type traffic for your IP. This will allow you to access your instance when we test.
   </image>
4. The third security group will be for our internal load balancer. Create this new security group and add an inbound rule that allows HTTP type traffic from your public instance security group. This will allow traffic from your web tier instances to hit your internal load balancer.
   </image>
5. The fourth security group we’ll configure is for our private instances. After typing a name and description, add an inbound rule that will allow TCP type traffic on port 4000 from the internal load balancer security group you created in the previous step. This is the port our app tier application is running on and allows our internal load balancer to forward traffic on this port to our private instances. You should also add another route for port 4000 that allows your IP for testing.
   </image>
6. The fifth security group we’ll configure protects our private database instances. For this security group, add an inbound rule that will allow traffic from the private instance security group to the MYSQL/Aurora port (3306).
   </image>

## Database Deployment - Part 2

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
