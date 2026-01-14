# Setting Up WordPress on AWS: A Step-by-Step Guide 

### Introduction
This project is about setting up WordPress on Amazon Web Services (AWS). In further sections of this documentation, we will be using separate EC2 instances for our web server and database, this is a setup that offers improved security and scalability.
In this documentation, you will be shown how I:

Configure EC2 instances with custom storage solutions using Logical Volume Manager (LVM)
Set up a LAMP (Linux, Apache, MySQL, PHP) stack on Red Hat Enterprise Linux
Install and configure WordPress to work with a remote database
Implement basic security measures to protect your setup


## Step 1: Preparing Your Web Server

We would start by setting up our web server. We will use a RedHat EC2 instance for this.

1. Launch a RedHat EC2 instance to serve as your Web Server. Create three 10GB volumes in the same Availability Zone (AZ) as your EC2 instance and attach them one by one.

![Web server volumes](images/createvolumes.JPG)

2. Now we will go into the Linux terminal! Connect to your instance using SSH:

3. Let's see what storage devices we're working with. Use `lsblk` to check the block devices:

```bash
lsblk
```
![List of block devices](images/lsblk.JPG)

You should see your new devices, probably named `nvme1n1`, `nvme2n1`, and `nvme3n1` because in my case I used `t3.micro` instance type

4. If you want to see your current disk space. Use `df -h`:

```bash
df -h
```

5. Now, let's partition those new disks using `parted`:

```bash
sudo parted /dev/nvme1n1 --script mklabel gpt mkpart primary ext4 0% 100%
sudo parted /dev/nvme2n1 --script mklabel gpt mkpart primary ext4 0% 100%
sudo parted /dev/nvme3n1 --script mklabel gpt mkpart primary ext4 0% 100%
```
![Partitioning nvme1n1](images/part1.JPG)
![Partitioning nvme2n1](images/part2.JPG)
![Partitioning nvme3n1](images/part3.JPG)

After partitioning, check your work with `lsblk`:

```bash
lsblk
```
![Updated block device list](images/3parts.JPG)

6. Time to install LVM (Logical Volume Manager). This will help us to manage our storage more flexibly:

```bash
sudo yum install lvm2 -y
```
![Installing LVM](images/install_lvm2.JPG)

7. Let's create physical volumes (PVs) from our partitions:

```bash
sudo pvcreate /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
sudo pvs
```
![Creating physical volumes](images/createpvs.JPG)

8. Next, we will group these PVs into a volume group (VG). we will call it `webdata-vg`:

```bash
sudo vgcreate webdata-vg /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
sudo vgs
```
![Creating volume group](images/createvgs.JPG)

9. Now for the logical volumes (LVs). We will create two: `apps-lv` for our website data, and `logs-lv` for our logs:

```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg
sudo lvcreate -n logs-lv -L 14G webdata-vg
sudo lvs
```
![Creating logical volumes](images/createlvs.JPG)

10. Let's double-check our setup:

```bash
sudo vgdisplay -v
lsblk
```
![Detailed LG display](images/lvdisplay.JPG)
![Updated block device list](images/lsblklv.JPG)

Now, let us format these volumes with ext4 filesystem:

```bash
sudo mkfs.ext4 /dev/webdata-vg/apps-lv
sudo mkfs.ext4 /dev/webdata-vg/logs-lv
```
11. We need directories to mount our volumes. Let us create them:

```bash
sudo mkdir -p /var/www/html
sudo mkdir -p /home/recovery/logs
```

12. Before we mount our logs volume, let's back up the existing log data:

```bash
sudo rsync -av /var/log /home/recovery/logs
```
13. Time to mount our volumes:

```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html
sudo mount /dev/webdata-vg/logs-lv /var/log
```

14. Now, let's restore our backed-up log files:

```bash
sudo rsync -av /home/recovery/logs/log/ /var/log
```
15. We want these mounts to persist after reboots, so let's update the `/etc/fstab` file:

```bash
sudo blkid
sudo vi /etc/fstab
```
16. Finally, let's test our configuration and make sure everything's working:

```bash
sudo mount -a
sudo systemctl daemon-reload
df -h
```
![Verifying setup](images/aftermount.JPG)

Great job! You've successfully set up your web server's storage. In the next step, we'll prepare the database server.

## Step 2: Preparing the Database Server

Now, let's set up our database server. The process is similar to the web server, but with a few key differences.

1. Launch another RedHat EC2 instance for your DB Server. Create and attach three 10GB volumes, just like before.

![DB server volumes](images/dbvolumes.JPG)

2. SSH into your new instance:

3. Check your block devices:

```bash
lsblk
```
![List of block devices](images/lsblkdb.JPG)

You should see your new devices, probably named `nvme1n1`, `nvme2n1`, and `nvme3n1` because in my case I used `t3.micro` instance type

4. If you want to see your current disk space. Use `df -h`:

```bash
df -h
```

5. Now, let's partition those new disks using `parted`:

```bash
sudo parted /dev/nvme1n1 --script mklabel gpt mkpart primary ext4 0% 100%
sudo parted /dev/nvme2n1 --script mklabel gpt mkpart primary ext4 0% 100%
sudo parted /dev/nvme3n1 --script mklabel gpt mkpart primary ext4 0% 100%
```
![Partitioning nvme1n1](images/part1.JPG)
![Partitioning nvme2n1](images/part2.JPG)
![Partitioning nvme3n1](images/part3.JPG)

After partitioning, check your work with `lsblk`:

```bash
lsblk
```
![Updated block device list](images/3parts.JPG)

6. Time to install LVM (Logical Volume Manager). This will help us to manage our storage more flexibly:

```bash
sudo yum install lvm2 -y
sudo lvmdiskscan
```
![Installing LVM](images/diskscandb.JPG)

7. Let's create physical volumes (PVs) from our partitions:

```bash
sudo pvcreate /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
sudo pvs
```
![Creating physical volumes](images/createpvsdb.JPG)

8. Next, we will group these PVs into a volume group (VG). we will call it `database-vg`:

```bash
sudo vgcreate database-vg /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
sudo vgs
```
![Creating volume group](images/createvgsdb.JPG)

9. Now for the logical volumes (LVs). We will create `db-lv` for our database data:

```bash
sudo lvcreate -n db-lv -L 20G database-vg
sudo lvs
```
![Creating logical volumes](images/createlvdb.JPG)

10. Let's double-check our setup:

```bash
sudo vgdisplay -v
lsblk
```
11. Format and mount the LV:

```bash
sudo mkfs.ext4 /dev/database-vg/db-lv
sudo mount /dev/database-vg/db-lv /db
```
12. Update `/etc/fstab` with the LV UUID for persistent mounting:

```bash
sudo blkid
sudo vi /etc/fstab
```
13. Test your configuration:

```bash
sudo mount -a
sudo systemctl daemon-reload
df -h
```
![Verifying DB setup](images/dbmount.JPG)

Excellent! Your database server is now set up and ready to go. Let's move on to installing WordPress.

## Step 3: Installing WordPress on the Web Server

Now that our servers are set up, it's time to install WordPress on our web server.

1. First, let's update our system:

```bash
sudo yum -y update
```

2. Install Apache and its dependencies:

```bash
sudo yum install wget httpd php-fpm php-json
```
3. Now, let's install PHP. We'll use the Remi repository for this:

```bash
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo yum install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-10.0.rpm
```
Check available PHP modules:
```bash
sudo dnf module list php
```
![Listing PHP versions](images/modulelistphp.JPG)

Enable PHP 8.2:
```bash
sudo yum module reset php
sudo yum module enable php:remi-8.2
```
Install PHP and its modules:
```bash
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```
Verify the PHP version:
```bash
php -v
```
Start and enable PHP-FPM:
```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm
```
4. Configure SELinux policies:

```bash
sudo chown -R apache:apache /var/www/html
sudo chcon -t httpd_sys_rw_content_t /var/www/html -R
sudo setsebool -P httpd_execmem 1
sudo setsebool -P httpd_can_network_connect=1
sudo setsebool -P httpd_can_network_connect_db=1
```
Restart Apache:
```bash
sudo systemctl restart httpd
```

5. Download and set up WordPress:

```bash
sudo mkdir wordpress && cd wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
cd wordpress/
sudo cp -R wp-config-sample.php wp-config.php
cd ..
sudo cp -R wordpress/. /var/www/html/
```
![WordPress login](images/wordpresssetup.JPG)

6. Now, let's set up MySQL on our DB Server:

Update the system:
```bash
sudo yum update -y
```
Install MySQL Server:
```bash
sudo yum install https://dev.mysql.com/get/mysql84-community-release-el10-1.noarch.rpm
sudo yum install mysql-community-server -y
```
Start and enable MySQL:
```bash
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```
![Starting MySQL on DB](images/mysqldinstall.JPG)

7. Configure the database for WordPress:

Secure your MySQL installation and get the preset root password to allow reset.
```bash
sudo mysql_secure_installation
sudo grep 'temporary password' /var/log/mysqld.log 
```
![Securing MySQL](images/createrootuser.JPG)

Create the WordPress database and user:
```bash
sudo mysql -u root -p

CREATE DATABASE wordpress_db;
CREATE USER 'wordpress'@'172.31.31.27' IDENTIFIED WITH mysql_native_password BY 'Admin123$';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress'@'172.31.31.27' WITH GRANT OPTION;
FLUSH PRIVILEGES;
show databases;
exit
```
![Creating WordPress database](images/createdbanduser.JPG)

Set the bind address in MySQL configuration:
```bash
sudo vi /etc/my.cnf
sudo systemctl restart mysqld
```
![Setting bind address](images/bindaddressdb.JPG)

8. Configure WordPress to connect to the remote database:

Open MySQL port 3306 on the DB Server EC2 instance, allowing access only from the Web Server's internal IP address.

![Opening MySQL port](images/updateddbSG.JPG)

Install MySQL client on the Web Server:
```bash
sudo dnf install https://dev.mysql.com/get/mysql84-community-release-el10-1.noarch.rpm -y
sudo yum install mysql-community-client -y
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```
![Installing MySQL client](images/msqlclient.JPG)

Edit the WordPress configuration:
```bash
cd /var/www/html
sudo vi wp-config.php
sudo systemctl restart httpd
```
![Editing wp-config](images/updatewpconfig.JPG)

Disable the Apache default page:
```bash
sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup
```

Test the connection to the DB Server:
```bash
sudo mysql -h 172.31.40.248 -u wordpress -p
show databases;
exit;
```
![Testing DB connection](images/remotedbconnect.JPG)

Now, access your WordPress site using your Web Server's public IP address. You should see the WordPress installation page.

![WordPress installed](./images/wordpressweb.JPG)
![WordPress login](./images/wordpresslogin.JPG)
![WordPress website](./images/webpage.JPG)

## Congratulations! 
You have successfully set up WordPress on AWS using separate Web and DB servers. Your WordPress site is now ready for content creation and customization.

## Challenges 
I faced a couple of challenges and here is how i troubleshouted them:
1. I used t3.micro hence my EBS were nvme and not Xen drivers as expected in t2.micro
2. My RHEL is version 10 and often got the Error: Problem: conflicting requests - nothing provides system-release(releasever) = 9 needed by remi-release-9.6-1.el9.remi.noarch from @commandline when installing from the Remi repository. To fix it, I ensured that the repo reads from 10.0 package variant from their website.
3. Installing mysql-server and mysql-client in RHEL 10. I used the mysql-community-server and mysql-community-client from their respective repos.
