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

- Deploy Database Layer
  - Subnet Groups
  - Multi-AZ Database

### Subnet Groups

1. Navigate to the RDS dashboard in the AWS console and click on Subnet groups on the left hand side. Click Create DB subnet group.
2. Give your subnet group a name, description, and choose the VPC we created.
   <image>
3. When adding subnets, make sure to add the subnets we created in each availability zone specificaly for our database layer. You may have to navigate back to the VPC dashboard and check to make sure you're selecting the correct subnet IDs.
   <image>

### Multi-AZ Database Deployment

1. Navigate to Databases on the left hand side of the RDS dashboard and click Create database.
2. We'll now go through several configuration steps. Start with a Standard create for this MySQL-Compatible Amazon Aurora database. Leave the rest of the defaults in the Engine options as default.
   <image>
   Under the Templates section choose Dev/Test since this isn't being used for production at the moment. Under Settings set a username and password of your choice and note them down since we'll be using password authentication to access our database.
   <image>
   Next, under Availability and durability change the option to create an Aurora Replica or reader node in a different availability zone. Under Connectivity, set the VPC, choose the subnet group we created earlier, and select no for public access.
   <image>
   Set the security group we created for the database layer, make sure password authentication is selected as our authentication choice, and create the database.
   <image>
3. When your database is provisioned, you should see a reader and writer instance in the database subnets of each availability zone. Note down the writer endpoint for your database for later use.
   <image>

## App Tier Instance Deployment - Part 3

In this section, we will create an EC2 instance for our app layer and make all necessary software configurations so that the app can run. The app layer consists of a Node.js application that will run on port 4000. We will also configure our database with some data and tables.

- Learning Objectives:
  - Create App Tier Instance
  - Configure Software Stack
  - Configure Database Schema
  - Test DB connectivity

### App Instance Deployment

1. Navigate to the EC2 service dashboard and click on Instances on the left hand side. Then, click Launch Instances.
2. Select the first Amazon Linux 2 AMI
3. We'll be using the free tier eligible <b>T.2 micro</b> instance type. Select that and click Next: Configure Instance Details.
4. When configuring the instance details, make sure to select to correct Network, subnet, and IAM role we created. Note that this is the app layer, so use one of the private subnets we created for this layer.
   <image>
5. We'll be keeping the defaults for storage so click next twice. When you get to the tag screen input a Name as a key and call the instance AppLayer. It's a good idea to tag your instances so you can easily keep track of what each instance was created for. Click Next: Configure Security Group.
   <image>
6. Earlier we created a security group for our private app layer instances, so go ahead and select that in this next section. Then click Review and Launch. Ignore the warning about connecting to port 22- we don't need to do that.
7. When you get to the Review Instance Launch page, review the details you configured and click Launch. You'll see a pop up about creating a key pair. Since we are using Systems Manager Session Manager to connect to the instance, proceed without a keypair. Click Launch.
   You'll be taken to a page where you can click launch instance, and you'll see the instance you just launched.

### Connect to Instance

1. Navigate to your list of running EC2 Instances by clicking on Instances on the left hand side of the EC2 dashboard. When the instance state is running, connect to your instance by clicking the checkmark box to the left of the instance, and click the connect button on the top right corner of the dashboard.Select the Session Manager tab, and click connect. This will open a new browser tab for you.

   <b>NOTE</b>: <i>If you get a message saying that you cannot connect via session manager, then check that your instances can route to your NAT gateways and verify that you gave the necessary permissions on the IAM role for the Ec2 instance. </i>

2. When you first connect to your instance like this, you will be logged in as ssm-user which is the default user. Switch to ec2-user by executing the following command in the browser terminal:
   ```
   sudo -su ec2-user
   ```
3. Let’s take this moment to make sure that we are able to reach the internet via our NAT gateways. If your network is configured correctly up till this point, you should be able to ping the google DNS servers:

   ```
   ping 8.8.8.8
   ```

   You should see a transmission of packets. Stop it by pressing Cntrl+C

   <b>NOTE:</b> <i>If you can’t reach the internet then you need to double check your route tables and subnet associations to verify if traffic is being routed to your NAT gateway! </i>

### Configure Database

1. Start by downloading the MySQL CLI:
   ```
   sudo yum install mysql -y
   ```
2. Initiate your DB connection with your Aurora RDS writer endpoint. In the following command, replace the RDS writer endpoint and the username, and then execute it in the browser terminal:

   ```
   mysql -h CHANGE-TO-YOUR-RDS-ENDPOINT -u CHANGE-TO-USER-NAME -p
   ```

   You will then be prompted to type in your password. Once you input the password and hit enter, you should now be connected to your database.

   <b>NOTE:</b> If you cannot reach your database, check your credentials and security groups.

3. Create a database called webappdb with the following command using the MySQL CLI:
   ```
   CREATE DATABASE webappdb;
   ```
   You can verify that it was created correctly with the following command:
   ```
   SHOW DATABASES;
   ```
4. Create a data table by first navigating to the database we just created:
   ```
   USE webappdb;
   ```
   Then, create the following transactions table by executing this create table command:
   ```
   CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL AUTO_INCREMENT, amount DECIMAL(10,2), description VARCHAR(100), PRIMARY KEY(id));
   ```
   Verify the table was created:
   ```
   SHOW TABLES;
   ```
5. Insert data into table for use/testing later:
   ```
   INSERT INTO transactions (amount,description) VALUES ('400','groceries');
   ```
   Verify that your data was added by executing the following command:
   ```
   SELECT * FROM trasactions;
   ```
6. When finished, just type exit and hit enter to exit the MySQL client.

### Configure App Instance

1. The first thing we will do is update our database credentials for the app tier. To do this, open the <b>application-code/app-tier/DbConfig.js</b> file from the github repo in your favorite text editor on your computer. You’ll see empty strings for the hostname, user, password and database. Fill this in with the credentials you configured for your database, the <b>writer</b> endpoint of your database as the hostname, and <b>webappdb</b> for the database. Save the file.

   <b>NOTE</b>: <i> This is NOT considered a best practice, and is done for the simplicity of the lab. Moving these credentials to a more suitable place like Secrets Manager is left as an extension for this workshop.</i>
   <image>

2. Upload the app-tier folder to the S3 bucket that you created in part 0.

3. Go back to your SSM session. Now we need to install all of the necessary components to run our backend application. Start by installing NVM (node version manager).
   ```
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
   source ~/.bashrc
   ```
4. Next, install a compatible version of Node.js and make sure it's being used
   ```
   nvm install 16
   nvm use 16
   ```
5. PM2 is a daemon process manager that will keep our node.js app running when we exit the instance or if it is rebooted. Install that as well.
   ```
   npm install -g pm2
   ```
6. Now we need to download our code from our s3 buckets onto our instance. In the command below, replace BUCKET_NAME with the name of the bucket you uploaded the app-tier folder to:
   ```
   cd ~/
   aws s3 cp s3://BUCKET_NAME/app-tier/ app-tier --recursive
   ```
7. Navigate to the app directory, install dependencies, and start the app with pm2.

   ```
   cd ~/app-tier
   npm install
   pm2 start index.js
   ```

   To make sure the app is running correctly run the following:

   ```
   pm2 list
   ```

   If you see a status of online, the app is running. If you see errored, then you need to do some troubleshooting. To look at the latest errors, use this command:

   ```
   pm2 logs
   ```

   <b>NOTE:</b> If you’re having issues, check your configuration file for any typos, and double check that you have followed all installation commands till now.

8. Right now, pm2 is just making sure our app stays running when we leave the SSM session. However, if the server is interrupted for some reason, we still want the app to start and keep running. This is also important for the AMI we will create:

   ```
   pm2 startup
   ```

   After running this you will see a message similar to this.

   ```
   [PM2] To setup the Startup Script, copy/paste the following command: sudo env PATH=$PATH:/home/ec2-user/.nvm/versions/node/v16.0.0/bin /home/ec2-user/.nvm/versions/node/v16.0.0/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user —hp /home/ec2-user
   ```

   <b>DO NOT run the above command,</b> rather you should copy and past the command in the output you see in your own terminal. After you run it, save the current list of node processes with the following command:

   ```
   pm2 save
   ```

### Test App Tier

Now let's run a couple tests to see if our app is configured correctly and can retrieve data from the database.

To hit out health check endpoint, copy this command into your SSM terminal. This is our simple health check endpoint that tells us if the app is simply running.

```shell
curl http://localhost:4000/health
```

The response should looks like the following:

```
"This is the health check"
```

Next, test your database connection. You can do that by hitting the following endpoint locally:

```bash
curl http://localhost:4000/transaction
```

You should see a response containing the test data we added earlier:

```json
{
  "result": [
    { "id": 1, "amount": 400, "description": "groceries" },
    { "id": 2, "amount": 100, "description": "class" },
    { "id": 3, "amount": 200, "description": "other groceries" },
    { "id": 4, "amount": 10, "description": "brownies" }
  ]
}
```

If you see both of these responses, then your networking, security, database and app configurations are correct.

<b>Congrats! Your app layer is fully configured and ready to go. </b>

## Internal Load Balancing and Auto Scaling

In this section,we will create an Amazon Machine Image (AMI) of the app tier instance we just created, and use that to set up autoscaling with a load balancer in order to make this tier highly available.

- <b>Learning Objectives:</b>
  - Create an AMI of our App Tier
  - Create a Launch Template
  - Configure Autoscaling
  - Deploy Internal Load Balancer

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
