# Simple Tableau Server Management in AWS

This project is meant to help Tableau Server admins who have deployed in AWS.  Tableau provides a few [quickstarts](https://aws.amazon.com/quickstart/architecture/tableau-server/), which help create a robust environment for your Tableau Server, but what if you just need something a bit simpler?  The idea behind this quick start is to help admins quickly spin up a new instances of their tableau server, import the configuration settings, restore data from the latest backup, and enable automated backups going forward.  It also lets you specify which version of Tableau Server you want to install, making upgrades seamless.  All of this can happen with just a few configuration steps in CloudFormation, and all without any downtime for your end users.

# How it works


# Instructions

## Step 1: Get your AWS environment setup

### IAM Role
In order for your new EC2 instance to complete it's setup, it needs permission to talk to the other AWS services.  It does this by using an IAM role, which defines what permissions the instance has.  In the AWS console, search for the *IAM* page and navigate to *Roles* in the left navigation.  From here, you can click the blue button to create a new role.  The first thing to do is select the type of entity that will be using the role.  Select EC2 as the service type, and then move onto Permissions.

The next page will ask what tasks the role is allowed to perform.  Make sure to add the following policies:

You can add tags to the role if needed, but otherwise just click next to review.  The last page will ask you to give this role a name, then click the create button.  You will need the name of your iam-role for the cloudformation template.

### S3 Bucket
We need a place to store our Tableau Server backups, and the best place to do this is in S3.  You can either use an existing bucket, or create a new bucket.  Either way, create a folder within your bucket to store your backups.  You will need this backup folder path as well as the bucket name for the cloudformation template.

An optional setting within S3 is to set a management rule for automatically cleaning up old backups.  You can use a _lifecycle rule_ to automatically delete objects older than _x_ days.

### SSL Certificate


### Load Balancers
There are 3 services you may want to expose on your Tableau Server: the Tableau Server web app, TSM, and repository.  In order to safely expose these, you will need a load balancer for each.  The Tableau Server and TSM web apps require application load balancers, and the repository needs a network load balancer.

To do this, search for EC2 service and navigate to the Load Balancer page using the left navigation.  Use the blue button to create a load balancer.  For the Tableau Server and TSM web apps, pick an Application Load Balancer.  For the repository, pick a Network Load balancer.  Next, you will need to specify the listener protocols, and availability zones for your load balancer.  For the Tableau Server and TSM web apps, you should have handlers for TCP port 80 and TLS port 443.  For repository, this should be just TCP port 8060.


Next, we need to configure the security settings.  Your certificate type should be set to _Choose a certificate from ACM_.  Make sure the right SSL certificate is selected from the dropdown list, and click next.

Security groups will depend on your AWS settings.  My setup includes a default group, which enables all communication on all ports to objects within the VPC.  I also have a second group, which allows traffic (web, ssl, db) to my Tableau Server.  In my example, I make sure both of these are checked for my load balancer.

The next step for configuring your load balancer is to setup a new target group.  This defines the group of EC2 instances that will receive incoming HTTP requests.  Give your group a name, and leave the rest of the settings.  The last step is to pick your EC2 instances as targets.  We can leave this blank, since the Cloudformation template will automatically add our EC2 instance to this list.

This process should be repeated for the Tableau Server web app, TSM web app, and Repository.  You will need the ARNs for all three load balancers, for the Cloudformation template

### Route53



## Step 2: Configure your Cloudformation template


## Step 3: Load the template in AWS Console


