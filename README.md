# Simple Tableau Server Management in AWS

This project is meant to help Tableau Server admins who have deployed in AWS.  Tableau provides a few [quickstarts](https://aws.amazon.com/quickstart/architecture/tableau-server/), which help create a robust environment for your Tableau Server, but what if you just need something a bit simpler?  The idea behind this quick start is to help admins quickly spin up a new instances of their tableau server, import the configuration settings, restore data from the latest backup, and enable automated backups going forward.  It also lets you specify which version of Tableau Server you want to install, making upgrades seamless.  All of this can happen with just a few configuration steps in CloudFormation, and all without any downtime for your end users.

# How it works
Cloudformation is a great way to instantiate new resources in AWS, based on a template of what you need and how they communicate with each other.  This project shows how you can use Cloudformation to automatically create a new EC2 instance, install Tableau Server, and restore from your latest backup.  There are several ways to setup your environment, but this approach was used to minimize downtime for end users.  We start with Route53 taking care of DNS and routing all incoming traffic to a series of Load Balancers.  This allows you to create friendly URLs for all the applications you want to provide access to.  For example you could have analytics.company.com go to your Tableau Server's web app, tsm.analytics.company.com go to your Tableau Server's TSM, and repo.analytics.company.com be the URL for your Tableau Server's postgres db.

![AWS Architecture](/screenshots/Overview.png)

In order to provide a seamless user experience, though, we need to differentiate between static resources (always existing) and dynamic resources (created by cloudformation).  In the above diagram Route53 record sets, load balancers, and S3 buckets are static.  This means they should always exist in your environment.  The EC2 instance that runs Tableau Server is created dynamically from our Cloudformation template, and gets added to load balancer's target groups.  The first time you run the cloudformation script, you'll be adding your first Tableau Server to the load balancer.  The second time you run the cloudformation script, it will create the new EC2 instance but wait until Tableau Server is installed and restored from the latest backup, before adding it to the load balancer and removing the old instance.  It's this process of waiting for the install/restore to complete that lets end users access the old instance before swapping in the new one.


# Instructions

## Step 1: Setup your AWS environment's static assets

### IAM Role
In order for your new EC2 instance to complete it's setup, it needs permission to talk to the other AWS services.  It does this by using an IAM role, which defines what permissions the instance has.  In the AWS console, search for the __IAM__ page and navigate to __Roles__ in the left navigation.  From here, you can click the blue button to create a new role.  The first thing to do is select the type of entity that will be using the role.  Select EC2 as the service type, and then move onto Permissions.
![Image of IAM wizard](/screenshots/iam-1.png)

The next page will ask what tasks the role is allowed to perform.  Make sure to add the following policies:
![Image of IAM Policies](/screenshots/iam-2.png)

You can add tags to the role if needed, but otherwise just click next to review.  The last page will ask you to give this role a name, then click the create button.  You will need the name of your iam-role for the cloudformation template.

### S3 Bucket
We need a place to store our Tableau Server backups, and the best place to do this is in S3.  You can either use an existing bucket, or create a new bucket.  Either way, search AWS for __S3__ and create a folder within your bucket to store your backups.  You will need this backup folder path as well as the bucket name for the cloudformation template.

An optional setting within S3 is to set a management rule for automatically cleaning up old backups.  You can use a _lifecycle rule_ to automatically delete objects older than _x_ days.
![Image of S3 lifecycle rule](/screenshots/s3.png)

### SSL Certificate
In order to ensure all traffic to your Tableau Server is secure, we need to use a SSL Certificate.  Search in AWS for __Certificate Manager__, in order to get a certificate.  This process will vary, depending on your AWS configuration and corporate policies, but there is a good guide [here](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html) for helping create your first certificate.  We will use the certificate you create, in when creating our load balancers.

### Load Balancers
There are 3 services you may want to expose on your Tableau Server: the Tableau Server web app, TSM, and repository.  In order to safely expose these, you will need a load balancer for each.  The Tableau Server and TSM web apps require _application load balancers_, and the repository needs a _network load balancer_.

To do this, search for __EC2__ service and navigate to the __Load Balancer__ page using the left navigation.  Use the blue button to create a load balancer.  For the Tableau Server and TSM web apps, pick an Application Load Balancer.  For the repository, pick a Network Load balancer.  Next, you will need to specify the listener protocols, and availability zones for your load balancer.  For the Tableau Server and TSM web apps, you should have handlers for TCP port 80 and TLS port 443.  For repository, this should be just TCP port 8060.
![Image of Load Balancer wizard](/screenshots/load-balancer-1.png)

Next, we need to configure the security settings.  Your certificate type should be set to _Choose a certificate from ACM_.  Make sure the right SSL certificate is selected from the dropdown list, and click next.

Security groups will depend on your AWS settings.  My setup includes a default group, which enables all communication on all ports to objects within the VPC.  I also have a second group, which allows traffic (web, ssl, db) to my Tableau Server.  In my case, I make sure both of these are checked for my load balancer.

The next step for configuring your load balancer is to setup a new target group.  This defines the group of EC2 instances that will receive incoming HTTP requests.  Give your group a name, and leave the rest of the settings.  The last step is to pick your EC2 instances as targets.  We can leave this blank, since the Cloudformation template will automatically add our EC2 instance to this list.
![Image of Target Group config](/screenshots/load-balancer-2.png)

This process should be repeated for the Tableau Server web app, TSM web app, and Repository.  You will need the ARNs for all three load balancers, for the Cloudformation template.  Once all three load balancers have been created, there is one more step for the Tableau Server and TSM web app load balancers.  For each, find the load balancer in the AWS console and click on the _Listeners_ tab.  You should see 2 listeners, one for HTTP (port 80) and one for HTTPS (port 443).  We want to automatically take traffic coming into port 80, and redirect it perminantly to port 443 to make it use SSL.  Select the *HTTP : 80* listener and click edit, then change the default action to redirect the incoming requests to port 443.
![Image of updating listeners](/screenshots/load-balancer-2.png)

### Route53
Now that our load balancers are ready, we just have to map them to friendly URLs.  Search AWS for __Route53__ and click on hosted zones, then click on your domain.  This process may also vary based on your AWS configuration and corporate policies, but in my case I have a domain for healthcare.tableau.com.  Within Route53, you can add _record sets_ which allow you to map URLs to objects in AWS (our load balancers).  For each of your load balancers, create a new record set of __Type A__ and select __Alias: Yes__.  You should now have a dropdown box to pick the target, with a list of your load balancers (and other objects in AWS).  Select the appropriate target alias and click save.  Now anyone who enters the url you specified, will get their traffic redirected to your load balancer.
![Image of record set](/screenshots/route53.png)

## Step 2: Configure your Cloudformation template
Now that all our static assets are ready, we need to make some minor tweaks to our cloudformation template.  Download the template file from this github repo, and open using a text editor.  If you are familiar with JSON, you should recognize that the template is just a large JSON object.  The important pieces to change are located in the Mappings section of this file.  
```json
"Mappings" : {
    "aws": {
      "LoadBalancerTargetGroup": {
        "tableauserver": "<your-tableau-server-load-balancer-target-group-arn>",
        "tsm": "<your-tsm-load-balancer-target-group-arn>",
        "repo": "<your-repository-load-balancer-target-group-arn>"
      },
      "s3": {
        "bucket": "<your-s3-bucket-name>",
        "backups": "/path/to/backups"
      },
      "ec2": {
        "ami": "ami-0c5204531f799e0c6",
        "securitygroups": "<security-group1>,<security-group2>,<etc>",
        "iaminstanceprofile": "<your-iam-role-name>"
      }
    },
    "tableau": {
      "Repository":{
        "user": "readonly",
        "password": "<repository-password>"
      },
      "AutomatedInstaller": {
        "url": "https://github.com/tableau/server-install-script-samples/raw/master/linux/automated-installer/packages/tableau-server-automated-installer-2019-1.noarch.rpm",
        "path" :"/opt/tableau/tableau_server_automated_installer/automated-installer.20191.19.0321.1733"
      },
      "Backups": {
        "script":"https://raw.githubusercontent.com/takashibinns/tableau-cloudformation-sample/master/backup.sh"
      }
    }
  },
  ```
This mapping object is broken down into AWS settings, and Tableau settings. Hopefully the AWS section is pretty straightforward, in terms of what is needed for each property.  The only thing to leave as-is, would be the ami.  This ami id is for Amazon Linux 2, which is the standard OS of EC2 instances.  You can change this, but you may need to tweak the cloudformation template if you pick an AMI that uses another package manager (instead of yum)

On the tabluea side, you should really only have to change your repository password.  This is the password assigned to the readonly user in Postgres, in case you access the repository in some workbooks.

## Step 3: Load the template in AWS Console
Steps 1 and 2 only need to be complete once.  This step is what you repeat, whenever you want to create a new instance of your Tableau Server.  In the AWS console, search for __CloudFormation__ and navigate to __Stacks__ using the left navigation.  Click the button to _Create Stack_, and select _With new Resources_.  Make sure _Template is Ready_ is selected at the top, and upload your customized cloudformation template.
![Cloudformation wizard - step 1](/screenshots/cloudformation-1.png)

This next page is where you enter the name of your stack (can be anything), and specify your parameters.  The default values on this page come from the _parameters_ section of your cloudformation template, so you may want to change what values are auto-populated here.  
![Cloudformation wizard - step 2a](/screenshots/cloudformation-2.png)

Once you set your defaults, you probably won't be changing much here excpect for the __InstallerURL__.  This is the link to download Tableau Server, and can be found on the [Tableau releases page](https://www.tableau.com/support/releases/server).  Copy the link for __linux__ than ends with __.rpm__, and paste it into the InstallerURL parameter.  This lets you specify which version of Tableau Server to install each time you run the script.  The reason we took this approach, is to make upgrades easier.  In this case, we install a fresh copy of Tableau Server and just restore all the server configurations and repository (data sources, workbooks, users, etc) once its done installing.
![Cloudformation wizard - step 2b](/screenshots/cloudformation-3.png)

The last two pages in the Cloudformation wizard can be left as-is, with no changes.  Once you're ready, click the orange _Create Stack_ button.  

Again, while this process is running (installation, configuration, restoring data) all users will leverage the existing Tableau Server.  Once the process is complete (will show as complete in the cloudformation event logs), users will be routed to the new tableau server.  When this happens, the old tableau server will be shut down and de-registered from the load balancer.  We don't terminate the instance, because in case anything goes wrong you can always manually start the old instance and add it back to the load balancers.

# Notes
If you're having trouble accessing Tableau Server from your web browser, here are some troubleshooting steps:

1. The first thing to check is if Route53 is handling DNS properly.   You can always test if Route53 is the issue, but just copy/pasting the DNS name of your load balancer into the browser. If the load balancer's DNS name resolves, then you know the problem is with Route53.
2. Next up is the load balancer.  If your load balancer's DNS name didn't resolve, check out it's target group.  You should be able to see a list of registered instances for the target group, and see if they are healthy or not.  If your instance is not healthy, its a problem on the tableau server side.  
3. If the instance is marked as healthy in the target group, its likely an issue with security groups.  Check the security groups of the load balancer and EC2 instance to make sure they allow traffic on ports 80/443.  There may also be ACLs at the VPC or Account level, which limit what kind of traffic is allowed.


If you're wondering how the EC2 instance gets setup with Tableau Server, the magic comes from the TableauServerEC2's _AWS::CloudFormation::Init_ section.  Here, we define a config set that performs a series of actions.  
1. Step one is to install all prerequisite packages on the new instance (like postgres odbc driver and the aws cli).  The automated backup script is also installed and scheduled, using CRON
2. Create a new local user, which is used to install Tableau Server
3. Download the most recent backups from our S3 bucket
4. Use Tableau's [Automated Installer](https://github.com/tableau/server-install-script-samples/tree/master/linux/automated-installer?_fsi=6G9o6EmY) to install tableau
5. Restore from the latest configuration and tsbak
6. Update the load balancers to user our new instance, and stop any old instances
