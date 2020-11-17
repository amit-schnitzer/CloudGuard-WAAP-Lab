# CloudGuard-WAAP-Lab
Check Point CloudGuard WAAP (Web Application and API protection) Hands-on Lab 

This is a practical, Hands-On experience with Check Point's CloudGuard WAAP product

## Pre-requisites 
Before you begin make sure you have:
  1. an AWS account that you are able to launch network and compute resources into 
  2. Account with Check Point Infinity Portal (portal.checkpoint.com)

## Lab Topology 
Our lab environment topology is pretty straight forward, it consists of a single VPC with 2 subnets in it (private and public)
Into that VPC, we'll deploy 2 instances. First one is a vulnerable website and the second one is our WAAP gw.
Both of our deployed instances will reside on the public subnet as this will allow us to test both scenarios (with and without protection)

![Topology](lab-topology.jpg)

## Preparing your Lab environment 
Log into your AWS account (https://console.aws.amazon.com/console/home) and browse to the VPC service page  
- Use “Launch VPC Wizard” and create a “VPC with a Single Public Subnet”
  - IPv4 CIDR block - 10.0.0.0/16 
  - VPC Name “WAAP LAB” (or whatever you feel like calling it)
  - Public subnet’s IPv4 CIDR -  10.0.0.0/24
  Leave the rest with default values
- Add a private subnet
  - Name Tag – Private Subnet
  - VPC – choose “WAAP LAB” VPC we’ve just created
  - IPv4 CIDR Block - 10.0.1.0/24
  Leave the rest with default values 
- Create an EC2 key pair
  - Key pair name – WAAP LAB
  - File Format ppk
  - Tags – you can leave empty
- Launch Amazon Linux 2 AMI (HVM), SSD Volume Type
  - EC2 Size – t2.micro is sufficient
  - Network – WAAP LAB VPC
  - Subnet – Public subnet
  - Auto Assign Public IP – enable
  - Network Interfaces --> Primary IP – 10.0.0.10
  - Security Group
    * Add port 7070 from 0.0.0.0/0
  - Click “launch” to finish the setup phase and launch our instance
  Leave the rest with default values and launch the instance
- Installing Docker on our Linux EC2 server 
  -	Run “sudo yum update –y”
  - Run “sudo yum install -y docker”
  - Run “sudo service docker start”
  - Run “sudo usermod -a -G docker ec2-user”
  - Logout and log back in
- Launch our vulnerable web site 
  - Run “docker run -d -p 7070:80 raesene/bwapp”
  - Run “docker ps” and make sure our container is running 
  -	Browse to http://<instance public IP>:7070/install.php to test our installation 
    - Click on “install”
    - Login 
- Generate WAAP agent token 
  - Login to Infinity Portal (portal.checkpoint.com) 
  - Create a Localhost Asset 
    - Go to ENVIRONMENT tab 
    -	Click on “New” to create a new asset 
    -	Choose a name for the new asset (i.e. Localhost)
    -	Under “application URL” type http://localhost and click on “+”
    -	Click the “Reverse Proxy” tab 
    -	Enter an upstream URL (i.e. http://127.0.0.1)
    -	Click “Save”
  - Go to “ENFORCEMENT” tab 
  - Under “Profiles” tab Click on “new” to create a new agent 
    -	Choose a name for the new agent (i.e. WAAPLAB)
    - Agent type – choose “Infinity Next Gateway”
    - Click on “Reverse Proxy” tab above 
    -	Make sure you check the previously create web asset (Localhost)
    Leave the rest with default values and click “Save”
  - Click on “Enforce”
  - Click on “Tokens” to generate a token (copy and save the token string as it will be used during gateway installation)
- Install Infinity GW 
  - Accept “Infinity Next Gateway” Terms and conditions in the following link https://aws.amazon.com/marketplace/pp?sku=cvxnu9a4ric9yop0tp0wdistb
  - Launch CloudGuard Infinity Next Gateway CFT into an existing VPC (#23 on the CFT list) from the following URL                         https://supportcenter.checkpoint.com/supportcenter/portal?eventSubmit_doGoviewsolutiondetails=&solutionid=sk111013
  - Choose “WAAP LAB” VPC 
  - Choose the public and private subnets accordingly 
  - Choose the key pair we previously created  
  - Enter Password Hash (i.e. “$1$IFkdjsxm$4rreJ1DM4TFCqJ/F4I2xs/” for Cpwins1!)
  - Infinity Next Agent Token (paste the token generated on step 7.f. above)
  Launch the installation and wait until CloudFormation finishes 

## Configuring WAAP protection
- Login to Infinity Portal (portal.checkpoint.com)
- Under “My Services”, click “Infinity Next”
- First lets, create an asset representing our web server 
  - Go to ENVIRONMENT tab and under “Assets” choose to create a new
  - Enter a name for the asset (i.e bwapp)
  - Under “Application URL” type is the following http://<gw public IP>
    Make sure to replace with the gateway’s public IP from the AWS console
  - Click on “Reverse Proxy” tab on the left 
  - Under “Upstream URL” enter the following http://10.0.0.10:7070
  If you’ve assigned a different IP to the web server, make sure to replace it accordingly
  - Click on “Enforce"
- Now let’s create a policy that will be pushed and enforced on our gateway 
  - Go to POLICY tab on the left 
  - Under Rules create on “New” to create a new rule
  - Under “Assets & Zones” click on “+” choose “Assets” and make sure you check the Asset we’ve created on step 2 above 
  - Under “Practices” click on “+” and choose “Web application protection”
  - Under “Triggers”, choose “Log”
  - Once done, click on “ENFORCE” to push policy on our gateway 

## Testing our configuration
Let’s test our configuration

### Use-case-1: SQL Injection Attack

- Launch SQL injection attack – NO CG-WAAP interference 
  Start by logging into our vulnerable website directly http://<web server public IP address>:7070/ using the following credentials bee/bug
  - On the top right corner choose “SQL Injection (GET/SEARCH)” and click on “Hack”
  - Now, in the search field, type “iron” for example and click on search 
  - Now, leave the search field empty and click on “search” 
  - Next, run the following “iron' or 1=1 -- “ and click “Search”
- Launch SQL Injection attack – CG-WAAP enabled
  - Login to Infinity Portal (portal.checkpoint.com)
  - Under “ENFORCEMENTS” section click “Profiles” on the left
  - Click “WAAPLAB” agent to see its configuration on the right
  - Under “REVERSE PROXY” remove “Localhost” tick and enable “bwapp” tick.
  -	On top middle click “ENFORCE” to push latest policy to our agent. 
  - Now, we’ll login the vulnerable website through our Infinity Next gateway http://<gateway public IP address>/login.php (same credentials as above) and repeat step 1a through     1d. As you can see, the actual attack is being blocked by our WAAP gateway

### Use-case-2: Directory Traversal Attack

- Launch Directory traversal attack (do this in both browser tabs and examine the differences) – NO CG-WAAP interference 
  - On the top right corner choose “Directory Traversal – Directories” and click on “hack” button 
  - Replace the word “documents” from the URL field with “/etc” and see what happens. 
- Test same steps with Infinity Gateway URL

### Use-case-3: PHP Code Injection Attack

- Launch PHP Code Injection (do this in both browser tabs and examine the differences)
  - On the top right corner choose “PHP Code Injection”  
  - Click on “message” and change the URL by replacing the word “test” with “phpinfo()”
  - Change phpinfo() with system(‘ls’) 
  - Change system(‘ls’) with exec('whoami');
- Test same steps with Infinity Gateway URL
- Inspect the logs on infinity portal 

