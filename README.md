# AWS-Cloud-for-Project-Set-Up-Lift-Shift



##  1. Clone This Repo and First Create Three Security Groups

![alt text](image.png)

 ####  1.1 Create for ELB

- Allow inbound rules for http,https from ipv4 and ipv6

####   1.2 Create for App

- Allow inbound rules for 8080 from ELB Security Group
- Allow inbound rules for ssh from your ip

####    1.3 Create for backend

- Allow inbound rules for 3306 from App Security Group
- Allow inbound rules for 11211 from App Security Group
- Allow inbound rules for 5672 from App Security Group
- Allow inbound rules for ssh from your ip
- Allow inbound rules for all traffic from App Security Group


##  2. Create Three Instances by using User Data Script

#### 2.1 User Data for mysql

```
#!/bin/bash
DATABASE_PASS='admin123'
sudo dnf update -y
sudo dnf install git zip unzip -y
sudo dnf install mariadb105-server -y
# starting & enabling mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
cd /tmp/
git clone -b awsliftandshift https://github.com/hkhcoder/vprofile-project.git
#restore the dump file for the application
sudo mysqladmin -u root password "$DATABASE_PASS"
sudo mysql -u root -p"$DATABASE_PASS" -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '$DATABASE_PASS'"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-project/src/main/resources/db_backup.sql
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
```

#### 2.2 Create EC2

- Security must be backend security group.
- Image Amazon 2023 AMI

![alt text](image-1.png)

#### 2.3  User Data for memcache

```
#!/bin/bash
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached
sudo memcached -p 11211 -U 11111 -u memcached -d
```

#### 2.4 Create EC2

- Security must be backend security group.
- Image Amazon 2023 AMI

![alt text](image-2.png)

#### 2.5  User Data for rabbitmq

```
#!/bin/bash
## primary RabbitMQ signing key
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc'
## modern Erlang repository
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key'
## RabbitMQ server repository
rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key'
curl -o /etc/yum.repos.d/rabbitmq.repo https://raw.githubusercontent.com/hkhcoder/vprofile-project/refs/heads/awsliftandshift/al2023rmq.repo
dnf update -y
## install these dependencies from standard OS repositories
dnf install socat logrotate -y
## install RabbitMQ and zero dependency Erlang
dnf install -y erlang rabbitmq-server
systemctl enable rabbitmq-server
systemctl start rabbitmq-server
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"

sudo systemctl restart rabbitmq-server
```


#### 2.6 Create EC2

- Security must be backend security group.
- Image Amazon 2023 AMI

![alt text](image-3.png)


#### 2.7  User Data for tomcat_ubuntu

```
#!/bin/bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-17-jdk -y
sudo apt install tomcat10 tomcat10-admin tomcat10-docs tomcat10-common git -y
```

#### 2.8 Create EC2

- Security must be app security group.
- Image Ubuntu 24

![alt text](image-4.png)


#### 2.9 Log in to APP EC2

-  check mariadb status

```
systemctl status mariadb
```

![alt text](image-5.png)


-  check mariadb db

```
mysql -u admin -p accounts
```

![alt text](image-6.png)


#### 2.10    Log in to Memcache EC2

-  check memcache status

```
systemctl status memcached
```

![alt text](image-7.png)


#### 2.11    Log in to Rabbitmq EC2

-  check rabbitmq status

```
systemctl status rabbitmq-server
```


![alt text](image-8.png)


## 3.Create Hosted Zone by using Route 53

#### 3.1 Use Private Hosted Zone

![alt text](image-9.png)
    
#### 3.2 Create Record by using private IP of EC2

![alt text](image-10.png)


## 4.Create S3 Bucket for Artifacts

#### 4.1 Create S3

![alt text](image-11.png)

#### 4.2 Create IAM user with S3 Permission

![alt text](image-12.png)

#### 4.3 Create role with S3 Permission

![alt text](image-13.png)

#### 4.4 Attach IAM role to EC2 App

![alt text](image-14.png)


#### 4.5 Update application.preperties file with Route 53 Records

![alt text](image-15.png)


#### 4.6 Check mvn version if not, please install 

```
mvn -v
sudo rm -rf /opt/apache-maven-3.9.7
cd /tmp
wget https://downloads.apache.org/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz
sudo tar -xzf apache-maven-3.9.9-bin.tar.gz -C /opt
sudo ln -sfn /opt/apache-maven-3.9.9 /opt/maven
source /etc/profile.d/maven.sh 2>/dev/null || source ~/.bashrc
mvn -v
```

![alt text](image-16.png)

![alt text](image-22.png)


#### 4.7 Install mvn

```
mvn install
```

![alt text](image-17.png)

#### 4.8 This is artifacts which must upload to S3

![alt text](image-18.png)


#### 4.9 Install AWS Cli and Configure IAM User

```
aws configure
```

#### 4.10 Upload Artifact to S3 by command

aws s3 cp target/vprofile-v2.war s3://tkk-vprofile-labs-artifacts 

![alt text](image-19.png)


#### 4.11 Install AWS Cli at EC2 APP


```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

```
sudo apt install unzip -y
```

```
unzip awscliv2.zip
```

```
sudo ./aws/install
```

```
aws --version
```


```
aws s3 cp s3://tkk-vprofile-labs-artifacts/vprofile-v2.war /tmp/
```

```
systemctl stop tomcat10.service
```

```
ls /var/lib/tomcat10/webapps/
```

Need to remove Root which is default folder

![alt text](image-20.png)

```
rm -rf /var/lib/tomcat10/webapps/ROOT/
```

Replace with Artifact

```
cp /tmp/vprofile-v2.war /var/lib/tomcat10/webapps/ROOT.war
```

```
systemctl start tomcat10.service
```


#### Add Security Group for EC2 App and Ping it

![alt text](image-21.png)

![alt text](image-30.png)

## 5. Create Loadbalancer and Target Group

#### 5.1 Create Target Group

- Choose 8080

![alt text](image-23.png)

![alt text](image-24.png)

![alt text](image-25.png)

![alt text](image-26.png)

#### 5.2 Create ALB

![alt text](image-28.png)

![alt text](image-27.png)

![alt text](image-29.png)


------
**_That's it!.You have finished How To Configure AWS Cloud for Project Set Up Lift and Shift.Special Thanks to Imran Teli_**
-----