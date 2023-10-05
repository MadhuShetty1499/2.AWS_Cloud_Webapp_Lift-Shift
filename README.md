# Project-2: AWS_Cloud_Webapp_Lift-Shift
AWS Cloud for Multi-Tier Web application setup using Lift &amp; Shift migration strategy

  About the Project:
  - Multi Tier web application stack [Vprofile].
  - Host & run on AWS cloud for production.
  - Lift & Shift migration strategy.

  Scenario:
  - Application services running on physical computers/VM.
  - Workload in your Data center.
  - Virtualization team, DC ops team, Monitoring team, Sys Admin team, etc are involved.

  Problem:
  - Complex management.
  - Scale up/down complexity.
  - Upfront cap ex & Regular Op ex.
  - Manual process.
  - Difficult to automate.
  - Time consuming.

  Solution:
  - Cloud setup.
  - Pay As U Go.
  - IAAS.
  - Flexibility.
  - Ease of Infra management.
  - Automation.

  Vprofile project Architecture:
  1. Nginx (Load Balancer)
  2. Tomcat
  3. RabbitMQ
  4. Memcached
  5. MySQL

  AWS services used:
  1. EC2 - VM for Tomcat, RabbitMQ, Memcached, MySQL.
  2. ELB - Nginx LB replacement.
  3. Autoscaling - Automation for VM scaling.
  4. S3 - Shared storage.
  5. Route53 - Private DNS service.
  6. Amazon Certificate Manager[ACM] - For securing website.

  Flow of Execution:
  1. Login to AWS.
  2. Create Key pair.
  3. Create Security Groups.
  4. Launch instance with User data [Bash scripts <PATH>]
  5. Update IP to name mapping in Route53.
  6. Build application from source code.
  7. Upload artifact to S3 bucket.
  8. Download artifact to Tomcat EC2 instance.
  9. Setup ELB with HTTPS [Cert from ACM].
  10. Map ELB endpoint to website name in your Domain service provider.
  11. Access the website and verify.
  12. Add tomcat EC2 instance to Autoscaling group for scalability.

  Prerequisites:
    - Chocolatey for windows (Package manager)
      + Open powershell as admin
        $ Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
      + Press 'Y' when prompted
      + verify $ choco --version

    - Java11
      $ choco install adoptopenjdk11 --version=11.0.11.9
      $ java -version

    - Maven3
      $ choco install maven --version=3.8.4
      $ maven -version

    - AWS CLI
      $ choco install python
      $ pip --version
      $ pip install awscli
      $ aws --version

   
  Detailed steps:
  - Create ACM certificate & attach it to Domain service provider & validate it.

  - Create security groups
    1. for Load balancer - Allow port 443 from any IP and also 80 from any Ip for debugging.
    2. for Tomcat - Allow port 8080 from Load balancer security group and also port 22 from myIP or anywhere for logging in.
    3. for backend services (RabbitMQ, MySQL, Memcached) - Allow port 3306 for MySQL, port 11211 for RabbitMQ & port 5672 for Memcached from Tomcat security group (Port's details are given in application.properties file in source code).
    4. in backend security group - Allow all traffic from its own security group (backend SG) for internal communication between backend services and also port 22 from myIP or anywhere for logging in.

  - Create key pair (pem)

  - Create EC2 instance for MySQL: 
    + name - db01 & some tags
    + AMI - CentOS 9
    + Instance type - t2.micro
    + select key pair
    + with default VPC, select backend security group
    + in advanced settings, add user data(bash script = mysql.sh)
    + launch

  - Create EC2 instance for Memcached: 
    + name - mc01 & some tags
    + AMI - CentOS 9
    + Instance type - t2.micro
    + select key pair
    + with default VPC, select backend security group
    + in advanced settings, add user data(bash script = memcache.sh)
    + launch  

  - Create EC2 instance for RabbitMQ: 
    + name - rmq01 & some tags
    + AMI - CentOS 9
    + Instance type - t2.micro
    + select key pair
    + with default VPC, select backend security group
    + in advanced settings, add user data(bash script = rabbitmq.sh)
    + launch 

  - Create EC2 instance for Tomcat: 
    + name - app01 & some tags
    + AMI - Ubuntu 22
    + Instance type - t2.micro
    + select key pair
    + with default VPC, select Tomcat security group
    + in advanced settings, add user data(bash script = tomcat.sh)
    + launch 

  - Wait for about 10 mins to bring up instances and provisioning services.

  Note: Those bash scripts contains installing and provisioning services. Go through the scripts <PATH> and also you can install manually by typing those commands into your respective EC2 instances.

  - Login to all instances and check the services are running:
    + to login - $ ssh -i <keypair> <user>@<publicIP> (user name and ssh command can be found by selecting required instance and click on connect => ssh client)
    + to check the services - $ systemctl status <servicename>
                              $ ss -plunt | grep <portnumber>
                              $ netstat -plant | grep <portnumber>
                              service name & its ports (can check in application.properties file)
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
    + Clone the source code to local - $ git clone <URL>
    + In VS code => ctrl + shift + p => search default terminal profile => select git bash => view => terminal
    + Go to cloned repo => in src/main/resources/application.properties => change host names to db01.vprofile.in, mc01.vprofile.in, rmq01.vprofile.in
    + In VS code terminal, check maven3, java11 and aws cli are installed.
    + Go to directory where pom.xml is present => build the artifact $ mvn install

  - Create IAM user:
    + IAM => user => s3admin => attach policy (s3 full access) => create => download csv file (keep it safe)
    + In git bash, configure aws credentials - $ aws configure (provide access key, secret key, region[us-east-1], output format[json] that are found in csv file)

  - Upload artifact to S3 bucket:
    + create bucket - $ aws s3 mb s3://<bucket_name> (Note: Bucket name should be unique or else it won't create)
    + Copy artifact to bucket - $ aws s3 cp target/vprofilev2.war s3://<bucket_name>

  - Deploy artifact on tomcat server
    + Login to tomcat instance - $ ssh -i <keypair> <user>@<publicIP>
    + Install aws cli - $ sudo apt update && sudo apt install awscli -y && aws --version
    + To access S3 bucket, the instance needs aws credentials to be configured as we did previously or else we can create a role and attach to the instance. Create role in IAM => s3 admin => attach policy (s3 full access) => create => go to ec2 => click on tomcat instance => instance settings => modify IAM role => select s3 admin role => attach.
    + Copy artifact from S3 to temp folder - $ aws s3 cp s3://<bucket_name> /tmp/
    + Stop tomcat service - $ systemctl stop tomcat9
    + Remove ROOT folder in tomcat directory - $ rm -rf /var/lib/tomcat9/webapps/ROOT
    + Copy & rename artifact as ROOT.war - $ cp /tmp/vprofilev2.war /var/lib/tomcat9/webapps/ROOT.war
    + Start the service - $ systemctl start tomcat9
    + Check application.properties file in /var/lib/tomcat9/webapps/ROOT/web-Inf/classes/application.properties

  - Create Load Balancer:
    + Create Target group => give name => port 8080 => health check (/login) => advanced setting => override port 8080 => healthy threshold 3secs => select tomcat instance => include as pending => create target
    + Create Application load balancer => internet facing => select all AZ's => select Load balancer security group => listener (80 & 443) forward to target group => select certificate  => create
    + Copy endpoint of load balancer => entry in domain service provider => CNAME => name (vprofile) => target (paste URL) => add
    + verify in the browser => https://vprofile.<domain>

  - Create Autoscaling Group:
    + Create AMI of tomcat instance => instance setting => images => create AMI
    + Create launch configuration => choose created AMI => t2.micro => attach s3 admin role => select tomcat SG => select keypair => create
    + Create autoscaling group => select launch configuration => select all subnets => Application load balancer => target group => health check ELB => min 1, desired 1, max 2 => tracking scaling policy => cpu utilization 50 => enable scale-in protection only if your instance should not get terminated => add SNS topic (Optional) => add tags => create
    + Autoscaling group creates new tomcat instance, so terminate tomcat instance which is created manually.

  - Validate:
    + verify in the browser => https://vprofile.<domain>
    + Login as admin_vp (username and password both) check the services

  Credits:
  <Github link>
