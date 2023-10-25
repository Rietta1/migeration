# MIGRATE A MULTI-TIER WEB APP TO AWS CLOUD USING LIFT AND SHIFT STRATEGY (Infrastructure AS A Service 'IAAS')

This is a multi tier Web Application Stack (mig) which we are going to run on AWS Cloud for Production using Lift and Shift Stategy. This project is done as Infrastructure as a Service(IAAS)

You would learn how to run application workload for production by using lift and shift strategy on aws

### The Scenario

- We have application services which are running on physical/virtual machines/

- All these work load running on your local data center

### Localhost Prerequisites 

- Github - choco/brew install git -y
- Maven - choco/brew install maven -y
- AWS CLI - choco/brew install awscli -y
- Vscode - choco/brew install vscode -y

### Architecture of AWS services for the project

1. 4 EC2 Instances, for tomcat(app01), rabbitmq(rmq01), mysql(db01), memcache(mc01).

2. ELB[load balancer] replacement for nginx LB

3. Autoscaling

4. S3 to store software artifact

5. EFS Storage

6. Route 53 for DNS zones

7. Amazon Certificate Manager (ACM) for https

8. IAM

9. EBS

10. 3 Security groups


### Architecture explanation on AWS

Users will access our website by using the URL and that URL will be pointed to an endpoint which its entry will be mentioned in godaddy DNS.

- Users of the app will access it throuhg a secured endpoint url(https://) and each request will be recieved by a load balancer which only accepts https traffic.

- The secured certficate ssl will be gotten from ACM

- The ALB will route the requst to tomcat instances(app)

- The Tomcat will the running on ec2 instances which will be managed by auto scaling group. Depending on the traffic load from users, the ec2 instances will be scaled out or scaled in.

- The Tomcat ec2 servers will be in a separete security group which will only allow traffic on port 8080 from only the application load balancer


- The application needs backend servers, which are mysql,memchace and rabbitmq

- Information of the backend server or the backend server IP will be placed in a route 53 private dns zone

- So Tomcat instance will access backend servers with the name which is gotten from route 53 private dns with the private ip addr will be gotten as well.

- The backend ec2 instances server will (rmq,mec,mysql) will all be in placed in another sercurity group




### FLOW OF EXECUTION

1. Login to AWS Account
2. Create Security groups
3. Create key pairs
4. Lunch Instances with user data [BASH SCRIPTS]
5. Update backend servers IP addr to name mapping in route 53
6. Build Application from source code
7. Upload to S3 bucket
8. Download artifact to Tomcat EC2 instance
9. Setup ELB with HTTPS [Cert from Amazon Certicate Manager]
10. Map ELB Endpoint to website name in Godaddy DNS
11. Verify
12. Build Autoscaling group for tomact instances

## Hands on

1. Login to AWS Account

2. Create Security groups : 
- First we create a security group for our load balancer: Go to EC2, security groups , click on create SG, give a name `mig-ELB-SG` , give description *Security group for mig prod Load Balancer*

- Give inbound rules of 
```
http , port: 80 , allow : anywhere
https , port : 443 , allow : anywhere
```

- Second create a security group for tomcat server: name `mig-app-SG` , give description *Security group for tomcat instances*

- Give inbound rules of

```
Custom TCP, port: 8080 , allow : only from SG of load balancer , description : allow traffic from ELB.

Custom TCP, port: 22 , allow : only from my ipp addr , description : allows ssh from me .

Custom TCP, port: 8080 , allow : only from my ipp addr , description : allows ssh from me .

```


- Third ,create a security group for the backend services(rmq,mc,mysql): name `mig-db-sg` , give description *Security group for mig backend serivces*

- Give inbound rules of 

```
Custom TCP, port: 22 , allow : only from my ipp addr , description : allows ssh from my ip .

MYSQL/Aurora, port: 3304 , allow : only from SG of app , description : allow 3306 traffic from application server.

Custom TCP, port: 11211 , allow : only from SG of app , description : allow tomcat to connect to memcache.

Custom TCP, port: 5672 , allow : only from SG of app , description : allows tomcat to connect to rabbitmq.

All taffic, port: ALL , allow : only from SG of app , description : allow internal traffic to flow on all ports.

*here all backend servers are to interact with each other so we open port `All`*

```

*note: you need to update your ip regularly if your public ip is changing regularly*

save it

3. Create key pairs : go to key pair, create keypair and neame ot `mig-prod-key` and download it to your system

4. Lunch Instances with user data [BASH SCRIPTS]
- go to this (github-link)[https://github.com/Rietta1/migeration.git] and clone it locally, run `git checkout aws-LiftAndshift` to change branch...

- go to the folder userdata and you will see the scripts to implement each service on the ec2 instance 

- lunch 4 ec2 instance , add the bash scripts to the lunch configuration and lunch 

`tomcat_ubuntu.sh` for app01 server. (ubuntu 22)
```
#!/bin/bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-11-jdk -y
sudo apt install tomcat9 tomcat9-admin tomcat9-docs tomcat9-common git -y
```

`mysql.sh` for db01 (centos 9)

```
#!/bin/bash
DATABASE_PASS='admin123'
sudo yum update -y
sudo yum install epel-release -y
sudo yum install git zip unzip -y
sudo yum install mariadb-server -y


# starting & enabling mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
cd /tmp/
git clone -b main https://github.com/hkhcoder/vprofile-project.git
#restore the dump file for the application
sudo mysqladmin -u root password "$DATABASE_PASS"
sudo mysql -u root -p"$DATABASE_PASS" -e "UPDATE mysql.user SET Password=PASSWORD('$DATABASE_PASS') WHERE User='root'"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-project/src/main/resources/db_backup.sql
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"

# Restart mariadb-server
sudo systemctl restart mariadb


#starting the firewall and allowing the mariadb to access from port no. 3306
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
sudo systemctl restart mariadb

```

`memcache.sh` for mc01 (centos 9)

```
#!/bin/bash
sudo dnf install epel-release -y
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached
firewall-cmd --add-port=11211/tcp
firewall-cmd --runtime-to-permanent
firewall-cmd --add-port=11111/udp
firewall-cmd --runtime-to-permanent
sudo memcached -p 11211 -U 11111 -u memcached -d

```

`rabbitmq.sh` for rmq01 (centos 9)

```
#!/bin/bash
sudo yum install epel-release -y
sudo yum update -y
sudo yum install wget -y
cd /tmp/
dnf -y install centos-release-rabbitmq-38
 dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
 systemctl enable --now rabbitmq-server
 firewall-cmd --add-port=5672/tcp
 firewall-cmd --runtime-to-permanent
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo systemctl status rabbitmq-server
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo systemctl restart rabbitmq-server

```

- verify what you have installed `curl http://169.254.169.254/latest/user-data` to get the details of the scripts installed

- Check your database installation

```
sudo -i
systemctl status mariadb
mysql -u admin -padmin123 accounts
show tables;

```
- check rabbitmq and tomcat server
```
ss -tunlp | grep 11211
systemctl status rabbitmq-server
systemctl status tomcat9
ls /var/lib/tomcat9
```


5. Update IP to name mapping in route 53

- go to route53 on aws, then create a private hosted zone named `mig.in` , add the vpc of the same region (us-east-1) where your servers are placed , click on create hosted zone

- create record, click simple routing, subdomain : db01 [mig.in] , A- records, place your db01 private ip address.

- Do same for mc01 and rmq01



6. Build Application from source code

- We would build the artifact locally on our vscode, push it to s3 bucket and then deploy it to our tomcat server

- add mig.in to src/main/resources/application.properties




# Database