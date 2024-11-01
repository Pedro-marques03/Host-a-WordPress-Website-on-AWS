![Alt text](2._Host_a_WordPress_Website_on_AWS.png)
---

# AWS Host a WordPress Website Project

This project demonstrates how to host a highly available, secure, and scalable WordPress website on AWS, leveraging key AWS services. Resources and deployment scripts used in this project are provided in this repository.

## Architecture

The project architecture consists of the following AWS resources:
- **VPC Configuration**: A custom VPC with public and private subnets distributed across two availability zones to ensure high availability and fault tolerance.
- **Internet Gateway**: Enables internet access for the instances within the VPC.
- **Security Groups**: Configured as network firewalls to control inbound and outbound traffic.
- **EC2 Instances**: Web servers deployed in private subnets for added security, running WordPress with Apache and MySQL.
- **Application Load Balancer**: Distributes incoming traffic across multiple EC2 instances within an Auto Scaling Group.
- **Auto Scaling Group**: Ensures elasticity by scaling instances up or down based on traffic and performance needs.
- **NAT Gateway**: Facilitates internet access for instances within private subnets.
- **EC2 Instance Connect Endpoint**: Provides secure, browser-based SSH access to EC2 instances.
- **Amazon Route 53**: Manages DNS records for the registered domain.
- **Amazon EFS**: Provides a shared file system for WordPress files.
- **Amazon RDS**: Managed MySQL database for persistent data storage.
- **AWS Certificate Manager**: Enables secure HTTPS connections.
- **SNS Notification**: Sends alerts for activity within the Auto Scaling Group.

## Prerequisites

- An AWS account with appropriate permissions.
- Registered domain for DNS and SSL setup.
- IAM role permissions to allow EC2, RDS, EFS, and other services.

## Deployment Instructions

### Step 1: VPC and Subnet Configuration
1. Set up a VPC with public and private subnets across two availability zones.
2. Attach an Internet Gateway to the VPC.
3. Deploy NAT Gateways in the public subnets to allow private subnets outbound internet access.

### Step 2: Security Configuration
1. Create Security Groups to control access to the EC2 instances, RDS, and EFS.
2. Allow HTTP and HTTPS traffic for public access to the WordPress site and SSH for EC2 instance access.

### Step 3: EC2 Instance Setup
Deploy EC2 instances using the following script to install necessary software, set up the web server, and configure WordPress:

```bash
#!/bin/bash
# Elevate to root
sudo su

# Update packages
sudo yum update -y

# Create HTML directory
sudo mkdir -p /var/www/html

# Define EFS DNS and mount it
EFS_DNS_NAME=fs-064e9505819af10a4.efs.us-east-1.amazonaws.com
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# Install Apache and PHP
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install MySQL and PHP extensions
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install -y mysql-community-server

# Start MySQL
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Set permissions for the web directory
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;

# Download and configure WordPress
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
```

### Step 4: Auto Scaling Group Launch Template
Use the following script for your Auto Scaling Group to ensure all instances are identically configured with Apache, PHP, and MySQL:

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP and extensions
sudo dnf install -y php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext php-json php-xml php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo php-openssl php-pdo php-tokenizer

# Configure EFS mount
EFS_DNS_NAME=fs-02d3268559aa2a318.efs.us-east-1.amazonaws.com
echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

# Set permissions and restart Apache
chown apache:apache -R /var/www/html
sudo service httpd restart
```

### Step 5: Load Balancer and Route 53
1. Configure the Application Load Balancer to balance traffic across EC2 instances in the Auto Scaling Group.
2. Create a Route 53 record to map your domain name to the Load Balancer.

### Step 6: SSL and Monitoring
1. Use AWS Certificate Manager to secure your site with HTTPS.
2. Configure SNS to send alerts for scaling events.

## Additional Notes

- **Database Configuration**: Update `wp-config.php` with your RDS database credentials.
- **Permissions**: Ensure `wp-content` is writable for file uploads and plugin installations.
- **Monitoring and Logging**: Enable CloudWatch for detailed metrics and alerts.
