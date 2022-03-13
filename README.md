# Firefly-III

### Firefly III setup requirements:
+ Install LEMP stack (Amazon EC2 instance) + SSL certs;
+ PHP-8.0 and additional modules;
+ Composer;
+ Database (Amazom RDS);
+ DNS (Amazon Route53);

### Setup Nginx
```
sudo apt update && sudo apt upgrade
sudo apt install nginx -y
```
### PHP 8.0 
```
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install -y php8.0  php8.0-{cli,zip,gd,fpm,common,mysql,zip,mbstring,curl,xml,bcmath,imap,ldap,intl}
```
### Configure php.ini
```
sudo vim /etc/php/7.3/fpm/php.ini
memory_limit = 512M
error_log = php_errors.log
```
### Configure Nginx
```
cd /etc/nginx/sites-enabled/
sudo mv default{,.bak}
sudo vim /etc/nginx/sites-enabled/firefly.conf
sudo systemctl restart nginx php8.0-fpm
```
### Composer
```
cd ~
curl -sS https://getcomposer.org/installer -o composer-setup.php
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```
### Install Firefly 
```
cd /var/www/html/
sudo composer create-project grumpydictator/firefly-iii --no-dev --prefer-dist firefly-iii 5.6.16
sudo chown -R www-data:www-data firefly-iii
sudo chmod -R 775 firefly-iii/storage
```

**Firefly requires its own database. In this case, I created a RDS MySQL database and allowed access to connect from the EC2 instance to the database in Security Group.**
```
To connect to created database: 
mysql -h {{RDS_ENDPOINT}} -u admin -p

CREATE DATABASE firefly_database;
CREATE USER 'fireflyuser'@'%' IDENTIFIED BY 'User_Password';
GRANT ALL PRIVILEGES ON firefly_database. * TO 'fireflyuser'@'%';
FLUSH PRIVILEGES;
exit;
```
 
**Then, I need to make changes at /var/www/html/firefly-iii/.env**
```
DB_CONNECTION=mysql
DB_HOST={{RDS_ENDPOINT}}
DB_PORT=3306
DB_DATABASE=firefly_database
DB_USERNAME=fireflyuser
DB_PASSWORD=User_Password
```
### Initialize the database
```
cd /var/www/html/firefly-ii
sudo php artisan migrate:refresh --seed
sudo php artisan firefly-iii:upgrade-database
sudo php artisan passport:install
```
### SSL Configuration 

**I decided to use Let’s Encrypt to secure access to Firefly-III. This way automatically appends secure directives for your firefly.conf.**
```
sudo apt install python3-certbot-nginx
sudo certbot --nginx -d firefly.pidumenk.ru -d www.firefly.pidumenk.ru (it could be @n26.com domain).
sudo systemctl reload nginx
```
### Route 53 

+ Link your domain (@n26.com) to EC2 instance in Route53.
+ [Check docs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-ec2-instance.html).
+ For instance, I configured my own domain [firefly.pidumenk.ru](https://firefly.pidumenk.ru)


# Access Management
For integration G-Suite and AWS could be used AWS SSO. It’s possible to add G Suite as an external identity provider.

More information: https://aws.amazon.com/ru/blogs/security/how-to-use-g-suite-as-external-identity-provider-aws-sso/

AWS SSO supports automatic user provisioning via the System for Cross-Identity Management (SCIM). However, this is not yet officially supported for G Suite custom SAML applications. The  user provisioning can only be done manually now. What it means is that AWS SSO has no way of automatically knowing all the usernames from Google Workspace. 

# CloudWatch Monitoring 

+ Create IAM role with **CloudWatchAgentPolicy**
+ Download CloudWatch Agent on EC2 instance and install it - [deb package](https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb)
+ Define your configuration file for CloudWatch and place it on the path: */opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/config.json
+ Activate CloudWatch agent: 
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/config.json
```
+ Create metric filter for log groups and custom alarms
+ Send alarms to SNS topic
