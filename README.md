# Project-2: AWS_Cloud_Webapp_Lift-Shift
In my previous project ([Project-1](https://github.com/MadhuShetty1814/1.Multi_Tier_Web_App_Local_Setup)), I accomplished the setup of a multi-tier web application locally. In my latest endeavor, I seamlessly migrated this application from my local VM setup to Amazon Web Services (AWS) using the 'lift and shift' strategy. This migration allowed me to leverage AWS's scalability and reliability while minimizing disruptions to the application's functionality. Key steps included provisioning resources, migrating data, and optimizing the setup, resulting in improved scalability, reliability, and cost-efficiency.

  ### About the Project:
  - Multi-Tier web application stack [Vprofile].
  - Host & run on AWS cloud for production.
  - Lift and shift migration strategy.

  ### Scenario:
  - Application services running on physical computers/VM.
  - Workload in your Datacenter.
  - Virtualization team, DC ops team, Monitoring team, Sys Admin team, etc. are involved.

  ### Problem:
  - Complex management.
  - Scale up/down complexity.
  - Upfront cap ex & Regular Op ex.
  - Manual process.
  - Difficult to automate.
  - Time-consuming.

  ### Solution:
  - Cloud setup.
  - Pay As U Go.
  - IAAS.
  - Flexibility.
  - Ease of Infra management.
  - Automation.

  ### Vprofile project Architecture:
  1. Nginx (Load Balancer)
  2. Tomcat
  3. RabbitMQ
  4. Memcached
  5. MySQL

  ### AWS services used:
  1. EC2 - VM for Tomcat, RabbitMQ, Memcached, MySQL.
  2. ELB - Nginx LB replacement.
  3. Autoscaling - Automation for VM scaling.
  4. S3 - Shared storage.
  5. Route53 - Private DNS service.
  6. Amazon Certificate Manager[ACM] - For securing website.

  ### Flow of Execution:
  1. Login to AWS.
  2. Create a Key pair.
  3. Create Security Groups.
  4. Launch instance with User data [Bash scripts <PATH>]
  5. Update IP to name mapping in Route53.
  6. Build application from source code.
  7. Upload artifact to S3 bucket.
  8. Download the artifact to the Tomcat EC2 instance.
  9. Set up ELB with HTTPS [Cert from ACM].
  10. Map the ELB endpoint to the website name in your Domain service provider.
  11. Access the website and verify.
  12. Add Tomcat EC2 instance to the Autoscaling group for scalability.

  ### Prerequisites:
  1. Chocolatey for Windows (Package manager)
      * Open Powershell as admin
        - `$ Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))`
      * Press 'Y' when prompted
      * verify $ choco --version

  2. Java11
      `$ choco install adoptopenjdk11 --version=11.0.11.9`
      `$ java -version`

  3. Maven3
      `$ choco install maven --version=3.8.4`
      `$ maven -version`

  4. AWS CLI
      ```
      $ choco install python
      $ pip --version
      $ pip install awscli
      $ aws --version
      ```
   
  ### Detailed steps:
  - Create an ACM certificate & attach it to the Domain service provider & validate it.

  - Create security groups
    1. For load balancer - Allow port 443 from any IP and 80 from any IP for debugging.
    2. for Tomcat - Allow port 8080 from the Load balancer security group and port 22 from my IP or anywhere for logging in.
    3. for backend services (RabbitMQ, MySQL, Memcached) - Allow port 3306 for MySQL, port 11211 for RabbitMQ & port 5672 for Memcached from the Tomcat security group (Port's details are given in the application.properties file in source code).
    4. in backend security group - Allow all traffic from its security group (backend SG) for internal communication between backend services and port 22 from myIP or anywhere for logging in.

  - Create key pair (pem)

  - Create EC2 instance for MySQL: 
    + name - db01 & some tags
    + AMI - CentOS 9
    + Instance type - t2.micro
    + Select key pair
    + With default VPC, select the backend security group
    + in advanced settings, add user data(bash script = mysql.sh)
    + launch

  - Create EC2 instance for Memcached: 
    + name - mc01 & some tags
    + AMI - CentOS 9
    + Instance type - t2.micro
    + Select key pair
    + With default VPC, select the backend security group
    + in advanced settings, add user data(bash script = memcache.sh)
    + launch  

  - Create EC2 instance for RabbitMQ: 
    + name - rmq01 & some tags
    + AMI - CentOS 9
    + Instance type - t2.micro
    + Select key pair
    + With default VPC, select the backend security group
    + in advanced settings, add user data(bash script = rabbitmq.sh)
    + launch 

  - Create EC2 instance for Tomcat: 
    + name - app01 & some tags
    + AMI - Ubuntu 22
    + Instance type - t2.micro
    + Select key pair
    + With default VPC, select Tomcat security group
    + in advanced settings, add user data(bash script = tomcat.sh)
    + launch 

  - Wait for about 10 minutes to bring up instances and provisioning services.

  Note: Those bash scripts contain installing and provisioning services. Go through the scripts <PATH> and also you can install manually by typing those commands into your respective EC2 instances.

  - Login to all instances and check the services are running:
    + to log in - `$ ssh -i <keypair> <user>@<publicIP>` (user name and ssh command can be found by selecting the required instance and clicking on connect => ssh client)
    + to check the services - `$ systemctl status <servicename>`
                              `$ ss -plunt | grep <portnumber>`
                              `$ netstat -plant | grep <portnumber>`
                              service-name & its ports (can check in application.properties file)
                                - mariadb (3306)
                                - memcached (11211)
                                - rabbitmq-server (5672)
                                - tomcat9 (8080)

  - Create Route53 (For private DNS):
    + Create zone => vprofile.in => private hosted zone => us-east-1(region) => create
    + Create record => simple routing => db01 => <private ip of db01 instance> => A record => create
    + Create record => simple routing => mc01 => <private ip of mc01 instance> => A record => create
    + Create record => simple routing => rmq01 => <private ip of rmq01 instance> => A record => create

  - Build & Deploy artifact:
    + Clone the source code to local - `$ git clone <URL>`
    + In VS code => ctrl + shift + p => search default terminal profile => select git bash => view => terminal
    + Go to cloned repo => in src/main/resources/application.properties => change host names to db01.vprofile.in, mc01.vprofile.in, rmq01.vprofile.in
    + In VS code terminal, check maven3, java11 and aws cli are installed.
    + Go to directory where pom.xml is present => build the artifact `$ mvn install`

  - Create IAM user:
    + IAM => user => s3admin => attach policy (s3 full access) => create => download CSV file (keep it safe)
    + In git bash, configure aws credentials - `$ aws configure` (provide access key, secret key, region[us-east-1], output format[json] that are found in CSV file)

  - Upload artifact to S3 bucket:
    + Create bucket - `$ aws s3 mb s3://<bucket_name>` (Note: Bucket name should be unique or else it won't create)
    + Copy artifact to bucket - `$ aws s3 cp target/vprofilev2.war s3://<bucket_name>`

  - Deploy artifact on Tomcat server
    + Login to tomcat instance - `$ ssh -i <keypair> <user>@<publicIP>`
    + Install aws cli - `$ sudo apt update && sudo apt install awscli -y && aws --version`
    + To access the S3 bucket, the instance needs AWS credentials to be configured as we did previously, or else we can create a role and attach it to the instance. Create a role in IAM => s3 admin => attach policy (s3 full access) => create => go to ec2 => click on tomcat instance => instance settings => modify IAM role => select s3 admin role => attach.
    + Copy artifact from S3 to temp folder - `$ aws s3 cp s3://<bucket_name> /tmp/`
    + Stop tomcat service - `$ systemctl stop tomcat9`
    + Remove ROOT folder in tomcat directory - `$ rm -rf /var/lib/tomcat9/webapps/ROOT`
    + Copy & rename artifact as ROOT.war - `$ cp /tmp/vprofilev2.war /var/lib/tomcat9/webapps/ROOT.war`
    + Start the service - `$ systemctl start tomcat9`
    + Check the application.properties file in /var/lib/tomcat9/webapps/ROOT/web-Inf/classes/application.properties

  - Create Load Balancer:
    + Create Target group => give name => port 8080 => health check (/login) => advanced setting => override port 8080 => healthy threshold 3secs => select tomcat instance => include as pending => create target
    + Create Application load balancer => internet facing => select all AZ's => select Load balancer security group => listener (80 & 443) forward to target group => select certificate  => create
    + Copy endpoint of load balancer => entry in domain service provider => CNAME => name (vprofile) => target (paste URL) => add
    + verify in the browser => `https://vprofile.<domain>`

  - Create Autoscaling Group:
    + Create AMI of tomcat instance => instance setting => images => create AMI
    + Create launch configuration => choose created AMI => t2.micro => attach s3 admin role => select tomcat SG => select keypair => create
    + Create autoscaling group => select launch configuration => select all subnets => Application load balancer => target group => health check ELB => min 1, desired 1, max 2 => tracking scaling policy => cpu utilization 50 => enable scale-in protection only if your instance should not get terminated => add SNS topic (Optional) => add tags => create
    + Autoscaling group creates a new tomcat instance, so terminate the tomcat instance which is created manually.

  - Validate:
    + verify in the browser => `https://vprofile.<domain>`
    + Login as admin_vp (username and password both) check the services

  ### Credits:
  <Github link>
