# Follow these installation steps on the AMI EC2 instance:

## Install PHP, Apache, and MySQL client:
https://docs.aws.amazon.com/linux/al2023/ug/ec2-lamp-amazon-linux-2023.html

1. To ensure that all of your software packages are up to date, perform a quick software update on your instance. This process might take a few minutes, but it is important to make sure that you have the latest security updates and bug fixes.
The -y option installs the updates without asking for confirmation. If you would like to examine the updates before installing, you can omit this option.
$ sudo dnf upgrade -y

2. Install the latest versions of Apache web server and PHP packages for AL2023.
[ec2-user ~]$ sudo dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel


3. Install the MariaDB software packages. Use the dnf install command to install multiple software packages and all related dependencies at the same time.
[ec2-user ~]$ sudo dnf install mariadb105-server


4. Start the Apache web server.
[ec2-user ~]$ sudo systemctl start httpd

5. Use the systemctl command to configure the Apache web server to start at each system boot.
[ec2-user ~]$ sudo systemctl enable httpd

6. Add your user (in this case, ec2-user) to the apache group.
[ec2-user ~]$ sudo usermod -a -G apache ec2-user


7. Change the group ownership of /var/www and its contents to the apache group.
[ec2-user ~]$ sudo chown -R ec2-user:apache /var/www

8. To add group write permissions and to set the group ID on future subdirectories, change the directory permissions of /var/www and its subdirectories.
[ec2-user ~]$ sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;


## Install git:
```
sudo dnf install git -y
```

## Add the following EC2 instance userdata to the Launch Template:

```
#!/bin/bash
sudo systemctl start httpd
cd /var/www/html
sudo git clone https://github.com/pritam554/three-tier-architecture-aws.git
sudo mkdir api
sudo mv /var/www/html/three-tier-architecture-aws/backend/api/* /var/www/html/api/
sudo rm -rf /var/www/html/three-tier-architecture-aws
sed -i 's/update-me-host/insert-your-database-host-here/g' /var/www/html/api/db_connection.php
sed -i 's/update-me-username/insert-your-database-username-here/g' /var/www/html/api/db_connection.php
sed -i 's/update-me-password/insert-your-database-password-here/g' /var/www/html/api/db_connection.php
```
