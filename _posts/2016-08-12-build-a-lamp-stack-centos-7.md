---
layout: post
title: How to build a LAMP (Linux, Apache, MySQL, PHP) Stack on CentOS 7
---

<strong>Step by step instructions on how to install a LAMP stack on CentOS 7 on AWS</strong>

This tutorial walks through the basics of setting up a standard LAMP stack on AWS. Running Linux, Apache, MySQL and PHP (LAMP) is one of the most popular configurations for dynamic web applications, utlitizing a powerful bundle of open source software.  By following the instructions in this guide, it should take about 15 minutes to have the LAMP server up and running.

Begin by creating an AWS EC2 t2.medium instance (or whatever size you wish) - it can always be resized before going into production usage.  Our preference in Linux flavor is CentOS 7, available through the AWS Marketplace, but these commands should work for any RPM-based system (RHEL, CentOS, Fedora).

To log into the newly minted server, please take note that along with the correct private key, you will also need to ssh in as the user `centos`.  For example:
{% highlight shell %}
# Insert your server IP or hostname below
ssh centos@123.45.6.789
{% endhighlight %}

For initial setup after provisioning the new server run the following commands:
{% highlight shell %}
# Update Yum
yum update

# Install wget
yum install wget

# Get the latest Epel and Remi Repos
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
wget http://rpms.remirepo.net/enterprise/remi-release-7.rpm

# Install the repos
rpm -Uvh remi-release-7.rpm epel-release-latest-7.noarch.rpm
{% endhighlight %}

Next, we'll install Apache.  Why Apache?  Because it will be the easiest to get you up and running - and this isn't the place to enter into the age-old Apache vs NGINX debate.
{% highlight shell %}
# Install Apache
yum install httpd

# Enable Apache to start on boot
systemctl enable httpd

# Start Apache
systemctl start httpd
{% endhighlight %}

Now its database install time!  Our preference is <a href="https://mariadb.org/about/" target="_blank">MariaDB</a>, which is a drop-in replacement for MySQL.  MariaDB is an open-source fork from MySQL 5.5, created by the original developers of MySQL in the wake of the acquisition by Oracle in 2009.  The high compatibility and exact matching of MySQL APIs and commands allows us to MariaDB seemlessly.

In order to get the most recent version of MariaDB, the repo needs to be installed manually.  To do so, go to MariaDB's <a href="https://downloads.mariadb.org/mariadb/repositories/" target="_blank">repo page</a> and navigate to your system.  In this case, it is located under CentOS > CentOS 7 > 10.1 [Stable]
{% highlight shell %}
# Create a new repo file
vi /etc/yum.repos.d/mariadb.repo
{% endhighlight %}

Paste the contents from the MariaDB repo page in the new file and save.  At the time of writing, the contents were the following:
{% highlight plaintext %}
# MariaDB 10.1 CentOS repository list - created 2016-06-07 01:17 UTC
# http://downloads.mariadb.org/mariadb/repositories/
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
{% endhighlight %}

Next its time to actually install MariaDB and because of the above steps, we can simply install the most recent version with yum. Yay!
{% highlight bash %}
# Install MariaDB
yum install MariaDB-server

# Enable MariaDB to start on boot
systemctl enable mariadb

# Start MariaDB
systemctl start mariadb

# Improve MariaDB installation security
mysql_secure_installation

# This is only necessary if an older major version of MariaDB or MySQL already existed on the server
# Use root password set in step above
mysql_upgrade -p
{% endhighlight %}

Now on to the installation of PHP 7. We've been migrating everything over to PHP 7 because of the optimizations - up to 30% - from some of our tests!
{% highlight shell %}
# Enable Remi's Repo (change `enabled=0` to `enabled=1`)
vi /etc/yum.repos.d/remi.repo

# Enable the PHP 7 repo (change `enabled=0` to `enabled=1`)
vi /etc/yum.repos.d/remi-php70.repo

# Install PHP and the PHP MariaDB/MySQL extension
yum install php php-mysql

# Restart Apache
systemctl restart httpd
{% endhighlight %}

To ensure that the system is working, create a new phpinfo file.
{% highlight shell %}
vi /var/www/html/test.php
{% endhighlight %}
and paste the following lines of code into the file.
{% highlight php %}
<?php
    phpinfo();
?>
{% endhighlight %}

Navigate to the IP or hostname of the server and there should be a PHP info page waiting.  If so, congratulations! Now you've got a fresh LAMP server ready the next amazing project coming down the pipeline.
