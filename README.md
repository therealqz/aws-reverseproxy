# aws-reverseproxy
An end to end AWS Cloud Solution to host 2 websites using reverse proxy. 

----

The Architecture

![Architecture](/images/Architecture.png)

Using the above architecture, I'll design a cloud solution to host 2 websites . I'll use AWS services like:

- VPC (Virtual Private Cloud)
- Route 53: 
- EC2
- RDS (Relational Database Management Service)
- Elastic File Service (EFS)
- EC2 AUTOSCALING
- Certificate Manager ( To provision, manage & deploy TLS/SSL certificates)
- ELB ( Elastic LoadBalancer)
- SNS

Explanation:
Requests from hosts on the internet to any of the 2 domains is received via Route53 DNS and routed to the VPC via the internet gateway to the External-ApplicationLB =which routes traffic to the 2 **AZ** using NginX reverse proxy residing in the Public subnets .Traffic is then routed to the 2 Apache Webservers(residing in the private subnet in 2 AZs) via the internal-ALoadBalancer. 

We'll introduce autoscaling to scale up and down the instances depending on CPU utilization when it hits the 90% threshold. The webservers are mounted to the EFS and we'll use RDS for our Relational DB service(free tier).
NOTE: For the RDS, I'll not use the KMS(Key management service for DB encryption) since this is a demo project.

I purchased a domain (davidemokpare.com) and I'm using Route53 DNS to manage the DNS and other mappings. I'll create 2 subdomains for the project - [DevOps-Tooling website](tooling.davidemokpare.com) & [wordpress website](wordpress.davidemokpare.com)

----------
-
#
Assign Certificate Manager to domain . When creating and Application LoadBalancer,you need to add a Certificate to it. All the sub-domains will be tied to the certificate. Use the  " .* " wildcard.
![](/images/)
---

Next Create the Amazon Elastic File system for NFS:

 Create NFS , Select the targets (DataLayer) and then create the Access points => What we specify to match the web servers. In this case ,one for Tooling webserver and other for Wordpress web server. If you use one for the 2, then it could overwrite the data and mess up the infrastructure. Create 2 access points so we don't have to mount our 2 web servers to a single access point. 



 ----

 Create **RDS** ( Amazon Relational Database Service) :  
 
 Before creating the RDS ,there are prerequisites;

1. Create a KMS ( Key Management Service ) to be used to encrypt the database ![KMS](/images/KMS-key-created.png). WILL NOT BE USED IN THIS PROJECT !!


On the **RDS** dashboard, Create a Subnet group and add 2 private subnets (Data Layer ) for the 2 AZs 
To ensure that the DB is highly available and has failover support,

I’ll configure a multi- AZ and create an RDS MYSQL 8 . I won't consider multi AZ’s since its highly unlikely that the whole region will fail (There's an AWS Solution for that too) .  

After creating the RDS DataBase subnet, create the RDS DataBase using the MySQL engine. Using the DataBase free tier template option means that you’ll not be able to . use KMS to encrypt your DB . RDS is very very expensive. I’ll use the free tier. Selecting the project VPC ,subnet group and Security group (ACS-DataLayer) where the database resides based on the Architecture. Database is now created .

 ![DataBase](/images/databaseCreation.png)  
 
  So far, we've created ,our   VPC, internet gateway, all the subnets and added them to the route table, we’ve created the NAT gateways, we've create the Aamazon Certificate ,  Amazon EFS, RDS,

---
---
Next Step

Create a **_Target group_**  ===> Create a **_Launchtemplate_** ⇒   Create the *Load Balancer. After that ,we can now create the AutoScaling group. Such that when creating our autoscaling group, it will use the launch template we have created and also use the Load Balancer we've created  to spin up the instances.

Note : The Launch template is like a recipe for creating an instance. You can use it individually or feed it to an Autoscaling group.

To create a Launch template , we need an AMI (Amazon Machine image) installed with our dependencies and programs based on our needs .

I’ll create 3 instances for the 3 AMI - Bastion, NGINX,  web-servers . 

I also created a Security group for the AMIs (ACS-ami) After creating the AMIs, we create the userdata  ![AMIs](/images/AMIs.png)
![AMI installation-webserver](/images/AMI-Installation-HistoryBastion.png)


Note: Create a project directory for config files and userdata for the 2 websites

---
Set up Compute Resources for NGINX
--

```
AMI-SetUp
Create an Ec2 instance based on CentOS AMI from the T2 micro family in a region closest to your customers or users.  Ensure it has the following software installed:
 
Python, ntp, vim,epel-release, htop, net-tools, wget , telnet, remi-repo,selinux config    …….
####
SET SELINUX POLICIES FOR NGINX PROXY SERVER
#
----

```
setsebool -P httpd_can_network_connect=1
setsebool -P httpd_can_network_connect_db=1
setsebool -P httpd_execmem=1
setsebool -P httpd_use_nfs 1

```

```
yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm

```
---

     This section below will install amazon efs utils for mounting the target on the Elastic file system  
---
---



```bash
git clone https://github.com/aws/efs-utils 

 cd efs-utils 

 yum install -y make
 
 yum install -y rpm-build  make rpm   
 
 yum install -y  ./build/amazon-efs-utils*rpm 
```
---
####

Set up self signed SSL cert for nginx instance.
#
___






```  
sudo mkdir /etc/ssl/private 

sudo chmod 700 /etc/ssl/private 


 openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/ACS.key -out /etc/ssl/certs/ACS.crt  


sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```


NOTE: You can use the private ipv4 DNS in the form entries when creating the ssl cert … Per AWS documentation, the LB does not validate the certificate but it needs to be installed . 

- 
How is the certificate used ? It’s used by specifying the path to the certificate in the NGINX configuration.


![nginX-Amisetup](/images/nginx-ami-setup1.png)

![nginx-Cert](/images/nginx-ami-cert.png)

----
----

###
Create an AMI out of the EC2 instance 
#


Prepare Launch template for NGINX (One Per Subnet )

Use the AMI to create a launch template  

Launch the instances into the public subnet 

Assign the appropriate security group


![SecurityGroup](/images/securitygroupsFinal.png)


configure userdata to update package repository with yum and install NGINX 
     
---
Configure Target Groups  

Select instances as the Target type

Select HTTPS protocol on secure port 443 using TLS 

Ensure the health check path is

 `/healthstatus`

Register Nginx instances as targets

Ensure the Health check passes for the target group

Configure Auto-Scaling for NginX

Select the right launch template 

Select the VPC

Select both public subnets

Enable ALB for the Auto Scaling Group

Select the Target group created before

Ensure you have health checks for both EC2 and ALB

Desired capacity is 2
Minimum capacity is 2
Maximum capacity is 4

Set Scale-out is CPU utilization reaches 90%

Ensure there is an SNS topic to send out notifications

----
---
####
Set up Compute Resources for Bastion
#

Provision the EC2 instances: 


Create an EC2 instance based on CentOS AMI in the same region and AZ as you have your Nginx proxy server.

Ensure it has the following: 

 ntp, net-tools, epel-repo , vim, python, wget, telnet , htop,mysql,chrony

Associate an Elastic IP address with each of the Bastion hosts

Create an AMI out of the EC2 instance


Prepare Launch Template for BASTION ( One per subnet )

Make use of the AMI to set up a launch template

Ensure the instances are launched in a public subnet

Assign appropriate security group

Update userdata to update yum package repository and install ansible and git

Configure Target Groups

Select instances as the target type

Ensure protocol is TCP on port 22

Register Bastion instances as target hosts

Ensure the target passes health checks

Configure Auto Scaling for Bastion:

Select the right launch templates

Launch into the right VPC

Select both public subnets

Enable ALB for the AutoScaling group

Select the target group created before

Ensure health check passes for both ALB and EC2

Desired Capacity is 2
Minimum capacity is 2
Maximum capacity is 4
Set SCale-out if CPU utilization reaches 90%

Ensure there is an SNS topic to send out scaling notifications


Configure compute resources for Web-Servers

Provision EC2-Instances for web servers.

 We need to create 2 launch templates for both the Tooling website and WordPress website.

Create an Ec2 instance for each of the websites using CentOS AMI ,in the same region and appropriate AZs
Ensure that the following software are installed: 

ntp, net-tools, php, vim, wget, htop, telnet, epel-release

----


**_WebServer AMI SetUp_**

```
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm

yum install wget vim python3 telnet htop git mysql net-tools chrony -y
```
Configure selinux policies for the web-servers and nginx  servers  

```
setsebool -P httpd_can_network_connect=1 

setsebool -P httpd_can_network_connect_db=1 

setsebool -P httpd_execmem=1 

setsebool -P httpd_use_nfs 1
```

---

###
Setting up self-signed certificate for the apache web server instance 
#

```
yum install -y mod_ssl  

openssl req -newkey rsa:2048 -nodes -keyout /etc/pki/tls/private/ACS.key -x509 -days 365 -out /etc/pki/tls/certs/ACS.crt  

vi /etc/httpd/conf.d/ssl.conf
```

---
####

NEXT: 

I have created 3 AMIs ![AMIs](/images/AMIs.png)

After creating the AMIs , I’ll create the target groups BEFORE creating the LoadBalancer

NOTE: I won't create a target group for the Bastion because it’s not going to be behind the LoadBalancer. The LB forwards traffic to the target groups

Target Groups created 

![TargetGroupsFinal](/images/target-groupsFinal.png)

The autoscaling group will launch instances to the target groups

Next => I’ll create the LB before creating the Launch Template

The 1st LB will be an internet facing LB and MAP the network to the appropriate proxy level subnets based on the project architecture 

The 2nd ALB will route traffic from proxy server to the web-servers in the private subnet 2 and 4 based on the project architecture .

 It resides in the private subnets 1 and 2 . Since we have 2 web servers targets ( wordpress & tooling ) , we must use 1 as default. 

 ![LoadBalancers](/images/LoadBalancerFinal.png)


After creating the LB ,we can configure an additional rule/conditions in the listeners section to also route tooling target traffic to the other (2nd)tooling target by reading the HOST-HEADERS on requests hitting the NginX reverse-proxy server.

Each rule can include one of each of the following conditions: host-header, path, http-request-method and source-ip. 

[INTERNALlb-CONDITIONS]       

 server [internalLB-Forwards]   [internalLB-rules-summary] 
[int-LB-summary]    [listener-edit]

Note: The private  subnets 1 and 2 subnet does not have a route to an internet gateway. 

This means that your load balancer will not receive internet traffic.

#

---

####
NEXT ⇒ Create Launch Templates and AutoScaling Groups
#
---

WordPress AZ1: 

Exists in private subnet 1 / Private subnet and attached to a NAT-gateway ( Not reachable from the internet) 

UserData 


```
#!/bin/bash

mkdir /var/www/

sudo mount -t efs -o tls,accesspoint=fsap-0fc3eb7ec3a48864d fs-012bbe2b767fb2d90:/ /var/www/  ( use the EFS access point for WordPress)

yum install -y httpd 

systemctl start httpd

systemctl enable httpd

yum module reset php -y

yum module enable php:remi-7.4 -y

yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json

systemctl start php-fpm

systemctl enable php-fpm

wget http://wordpress.org/latest.tar.gz

tar xzvf latest.tar.gz

rm -rf latest.tar.gz

cp wordpress/wp-config-sample.php wordpress/wp-config.php


mkdir /var/www/html/

cp -R /wordpress/* /var/www/html/

cd /var/www/html/

touch healthstatus

sed -i "s/localhost/database-1.c6qmnsitgq99.us-east-2.rds.amazonaws.com/g" wp-config.php  (use the RDS DataBase AccessPoint )

sed -i "s/username_here/admin/g" wp-config.php 

sed -i "s/password_here/admin12345/g" wp-config.php 

sed -i "s/database_name_here/wordpressdb/g" wp-config.php 

chcon -t httpd_sys_rw_content_t /var/www/html/ -R

systemctl restart httpd
```




#### Nginx Launch Template #

Resides in public subnet1 & 2
⇒  

```
#!/bin/bash

yum install -y nginx

systemctl start nginx   

systemctl enable nginx

git clone https://github.com/therealqz/ACS-project-config.git

mv /ACS-project-config/reverse.conf /etc/nginx/

mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf-distro

cd /etc/nginx/

touch nginx.conf

sed -n 'w nginx.conf' reverse.conf

systemctl restart nginx

rm -rf reverse.conf

rm -rf /ACS-project-config
```


####
Bastion Launch Template
#

⇒ UserData  
```
#!/bin/bash 

yum install -y mysql 

yum install -y git tmux 

yum install -y ansible
```

#### Tooling Launch Template #

Tooling AZ2

⇒ Create Template .Tooling instances exist in private subnet2 [ ] 

UserData

```
#!/bin/bash

mkdir /var/www/

sudo mount -t efs -o tls,accesspoint=fsap-0ec579e62993ded00 fs-012bbe2b767fb2d90:/ /var/www/

yum install -y httpd 

systemctl start httpd

systemctl enable httpd

yum module reset php -y

yum module enable php:remi-7.4 -y

yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json

systemctl start php-fpm

systemctl enable php-fpm

git clone https://github.com/therealqz/tooling-1.git

mkdir /var/www/html

cp -R /tooling-1/html/*  /var/www/html/

cd /tooling-1

mysql -h database-1.c6qmnsitgq99.us-east-2.rds.amazonaws.com -u admin -p toolingdb < tooling-db.sql

cd /var/www/html/

touch healthstatus

sed -i "s/$db = mysqli_connect('mysql.tooling.svc.cluster.local', 'admin', 'admin', 'tooling');/$db = mysqli_connect('database-1.c6qmnsitgq99.us-east-2.rds.amazonaws.com', 'admin', 'admin12345', 'toolingdb');/g" functions.php

chcon -t httpd_sys_rw_content_t /var/www/html/ -R

systemctl restart httpd

```

All launchTemplates have been created … 

####
CREATE AutoScalingGroup
#

⇒ Bastion, NGINX, Web Servers- 

Before creating auto scaling groups for webservers, 

I’ll login to the RDS host and create 2 DBs ( toolingdb & wordpressdb) . 

Use eval -s to copy your keys to the bastion host:

---

[Bastion-Login-RDS]

[ All-Auto Scaling groups] 

Delete the AMI instances as they are no longer needed

NEXT ⇒ CREATE AUTOSCALING GROUP 

FOR WordPress & Tooling Targets. 

First, check health status of targets

Create autoscaling group for wordpress and tooling 

  ![AutoScalingGroups](/images/Autoscaling-GroupFinal.png)

Create records in route53 for tooling.davidemokpare.com , www.tooling.davidemokpare.com and wordpress.davidemokpare.com

www.wordpress.davidemokpare.com

Configure it to route traffic to the LoadBalancer



![Route53](/images/route53Final.png)


![TargetGroupsFinal](/images/target-groupsFinal.png)


![Tooling website](/images/tooling-COMPLETED.png)

![wordpress-website](/images/wordpress%20Completed.png)

Find my Project configuration files here :  

https://github.com/therealqz/ACS-project-config

