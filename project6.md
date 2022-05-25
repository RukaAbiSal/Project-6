# WEB SOLUTION WITH WORDPRESS.
- The task of the project is to prepare storage infrastructure on two Linux servers and implement a basic web solution using WordPress.
- The project consist of two parts
- i. Configure storage subsystem for Web and Database servers based on Linux OS.
- ii. Install WordPress and connect it to a remote MySQL database server.  
- In this project , we will ensure that the disks used to store files on the Linux servers are adequately partitioned and managed through programs such as gdisk and LVM respectively.

## INSTALLATIONS AND STEPS REQUIRED.
- In this project we wil be using 'Redhat' (fully compatible derivative - CENTOS).
- Create and configure two linux-based virtual servers (EC2 instances in AWS): Webserver and Database.
- Create 3 Volumes for the Webserver and Database instances respectively with availabilty zone same with what is showing in the instances.
<img width="898" alt="instances" src="https://user-images.githubusercontent.com/104162178/170193362-12eaa52e-3ffb-4c4c-a405-e5a68330ce94.PNG">
<img width="592" alt="creatingvolumes" src="https://user-images.githubusercontent.com/104162178/170194126-6d76fe94-0cff-404c-a0c5-0862da8b439d.PNG">

- Attach volume to the Volumes created for the Webserver EC2 and Database EC2 ensuring that in the instance slot, you select the instance associated with each instance (Webserver OR Database).
- Open the linux terminal to begin configuration, use command `lsblk` to inspect what block devices are attached to both Webserver and Database EC2.
- 
<img width="305" alt="volumes" src="https://user-images.githubusercontent.com/104162178/170201955-7266ad9c-ed84-454b-96bf-545ce8ea34b6.PNG">

- Use gdisk to create a single partition on each of the 3 disks `sudo gdisk /dev/xvdf /dev/xvdg /dev/xvdh`.
- Use lsblk utility to view the newly configured partition on each of the 3 disks. Check screenshot below.
<img width="323" alt="PartitionsCreated" src="https://user-images.githubusercontent.com/104162178/170201128-bc3f1bd3-1ef8-4145-91ae-e2803e20556d.PNG">

- Install lvm2 package using `sudo yum install lvm2 -y` 


- Run pvcreate on each of the 3 disks as physical volumes to used by LVM `sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`.
- Verify phyical volume has been created successfully running `sudo pvs`.

- Run vgcreate command to add all 3 physical volumes to a volume group (VG), namely webdata-VG and database-VG respectively for both instances. 
- The commands are `sudo vgcreate webdata-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1`, `sudo vgcreate database-vg /dev/xvdh1 /dev/xvdg1 /dev/xvdf1` respectively.
- Verify that volume group has been created successfully by running `sudo vgs`.
<img width="369" alt="diskgroup" src="https://user-images.githubusercontent.com/104162178/170204232-8077b5cb-e73b-4a75-8a94-692eb90d2aa7.PNG">

- Use lvcreate utility to create 2 logical volumes for Webserver EC2 and 1 logical volume for Database EC2 respectively.
- For Webserver EC2, the 2 logical volumes are apps-lv (storing data for website) and logs-lv (storing data for logs). Half of the physical volumes will be allocated to each logical volumes.
- The commands are `sudo lvcreate -n apps-lv -L 14G webdata-vg`  and `sudo lvcreate -n logs-lv -L 14G webdata-vg` respectively.
- For Database EC2, only one logical volume is created `sudo lvcreate -n db-lv -L 20G database-vg`.
- Verify logical volume has been created by running sudo lvs.
<img width="710" alt="Logicalvolumescreated" src="https://user-images.githubusercontent.com/104162178/170202719-e1a039bb-23c4-4b1a-8dce-bcf6fde2d3c4.PNG">

- Format logical volumes with ext4 filesystem for both Webserver and Database EC2.
- For Webserver EC2 run `sudo mkfs.ext4 /dev/webdata-vg/apps-lv` and `sudo mkfs.ext4 /dev/webdata-vg/logs-lv`, while for Database run `sudo mkfs.ext4 /dev/database-vg/db-lv`.
- In Webserver EC2, create directory to store website files `sudo mkdir -p /var/www/html` and store backup of log data `sudo mkdir -p /home/recovery/logs`.
- In Webserver EC2, mount apps-lv (logical volume) to directory for storing websites files `sudo mount /dev/webdata-vg/apps-lv /var/www/html/`.
- In Webserver EC2, use rsync utility to backup all files in the log directory /var/log into /home/recovery/logs (Required before mounting the file system).
- Run command `sudo rsync -av /var/log/. /home/recovery/logs/`.
- Mount logs-lv (logical volume) to directory /var/log, `sudo mount /dev/webdata-vg/logs-lv /var/log`.
- All existing data in the /var/log will be deleted when logs-lv (logical volume) is mounted as such that is why created backup file using the rsync utilty for all the contents in the folder.
- Restore log files back into /var/log directory `sudo rsync -av /home/recovery/logs/. /var/log`.
- Update /etc/fstab file to ensure the mount configuration will persist after restart of the server.
- Run sudo blkid to copy the UUID of the device for both Webserver and Database 
<img width="941" alt="UUID" src="https://user-images.githubusercontent.com/104162178/170204293-3747e378-d2a6-4714-83e2-7acfecf30b72.PNG">

- Run sudo vi /etc/fstab to update the content with the UUID information.
- Test the configuration `sudo mount -a`, if no error message displays then it implies it installations went smoothly.
- Reload the daemon `sudo systemctl daemon-reload`.
- Verify your setup by running `df -h`.
<img width="548" alt="AfterFstab" src="https://user-images.githubusercontent.com/104162178/170206142-ce9d66b7-3009-4e58-9eda-9e8f5d902179.PNG">

- Install WordPress, Apache and its dependecies on Webserver EC2 `sudo yum -y update` , `sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json`.
- Start Apache sudo `systemctl enable httpd` , `sudo systemctl start httpd`.
- Install PHP and its dependencies.
```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```

- Restart Apache `sudo systemctl restart httpd`.
- Download wordpress and copy content to /var/www/html
- Create wordpress directory and cd into it `mkdir wordpress && cd wordpress`.
- Download wordpress inside wordpress folder `sudo wget http://wordpress.org/latest.tar.gz`, extract the file `sudo tar xzvf latest.tar.gz`.
- Make a copy of wp-config-sample.php as wp-config.php in the same wordpress folder `cp wordpress/wp-config-sample.php wordpress/wp-config.php`.
- cd into wordpress folders `cd wordpress/wordpress/`
- Copy the content inside wordpress into /var/www/html/ `sudo cp -R wordpress/. /var/www/html/`.
- Configure SELinux Policies in the Webserver SE2 `sudo chown -R apache:apache /var/www/html`, `sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R`, `sudo setsebool -P httpd_can_network_connect=1`.
- Install MySQL in Database Server EC2 `sudo yum update`, `sudo yum install mysql-server`.
- Verify service is up and running `sudo systemctl status mysqld`.
- Run the security script `sudo mysql_secure_installation` which helps to prepare mysql server instance.
- Run MySQL `sudo mysql -u root -p`
- Configure the database to work with WordPress.
-Run `select user, host from mysql.user` 
<img width="291" alt="Userhost" src="https://user-images.githubusercontent.com/104162178/170262164-e80deedb-1154-4169-9bbf-b875766de36a.PNG">

- In the Database Server EC2 instance, edit my.cnf.d file in /etc folder to include bind-address 0.0.0.0 sudo vi /etc/my.cnf.d and restart mysql `sudo systemctl restart mysqld`.
- In the Webserver EC2 instance, edit wp-config.php in /var/www/html/ folder to include Database Server MySQL information `sudo vi wp-config.php`.

<img width="643" alt="wpconfig" src="https://user-images.githubusercontent.com/104162178/170264135-fedee241-7765-4d5d-bb20-5938f2e17437.PNG">

- Restart httpd `sudo systemctl restart http`.
- Disable the default page of Apache `mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup`.
- Open MYSQL port 3306 on the Database Server EC2, for extra security allow access to the Database server ONLY from the WebServer's IP address.


- To confirm if Webserver EC2 can communicate with Database EC2, run `sudo mysql -h 172.31.23.211 -u myuser -p`.
- 
<img width="631" alt="Connectfromweb" src="https://user-images.githubusercontent.com/104162178/170265557-e243e25a-dc9b-444a-98a2-7a73479854e3.PNG">

-  Enable TCP port 80 in Inbound Rules configuration for your Web Server EC2 (enable from everywhere 0.0.0.0/0 or from your workstation’s IP).
-  Try to access from the browser the link to your WordPress




