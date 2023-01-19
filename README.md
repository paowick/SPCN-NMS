# Install LibreNMS
 ```
$ apt install software-properties-common
$ add-apt-repository universe
$ add-apt-repository ppa:ondrej/php
$ apt update
$ apt install acl curl fping git graphviz imagemagick mariadb-client mariadb-server mtr-tiny nginx-full nmap php-cli php-curl php-fpm php-gd php-gmp php-json php-mbstring php-mysql php-snmp php-xml php-zip rrdtool snmp snmpd whois unzip python3-pymysql python3-dotenv python3-redis python3-setuptools python3-systemd python3-pip
 ```
## -Add librenms user
 ```
$ useradd librenms -d /opt/librenms -M -r -s "$(which bash)"
 ```

## -Download LibreNMS
 ```
$ cd /opt
$ git clone https://github.com/librenms/librenms.git
 ```

## -Set permissions
 ```
$ chown -R librenms:librenms /opt/librenms
$ chmod 771 /opt/librenms
$ setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
$ setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
 ```



## -Install PHP dependencies
 ```
$ su - librenms
$ ./scripts/composer_wrapper.php install --no-dev
$ exit
 ```
 ```
$ wget https://getcomposer.org/composer-stable.phar
$ mv composer-stable.phar /usr/bin/composer
$ chmod +x /usr/bin/composer
 ```
## -Set timezone
 ```
$ vim /etc/php/8.1/fpm/php.ini
 ```
 ```
[Date]
; Defines the default timezone used by the date functions
; https://php.net/date.timezone
date.timezone = Asia/Bangkok   <---- here!!!
 ```
 ```
$ vim /etc/php/8.1/cli/php.ini
 ```
```
[Date]
; Defines the default timezone used by the date functions
; https://php.net/date.timezone
date.timezone = Asia/Bangkok   <----  hrer!!!
```
## -Configure MariaDB
```
$ vim /etc/mysql/mariadb.conf.d/50-server.cnf
```
```
tmpdir                  = /tmp
lc-messages-dir         = /usr/share/mysql
innodb_file_per_table=1     <---- here!!!
lower_case_table_names=0    <---- here!!!
#skip-external-locking
```
```
$ systemctl enable mariadb
$ systemctl restart mariadb
```
```
$ mysql -u root
```

```
$ CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
$ CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'password';
$ GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
$ FLUSH PRIVILEGES;
$ exit
```
## -Configure PHP-FPM
```
$ cp /etc/php/8.1/fpm/pool.d/www.conf /etc/php/8.1/fpm/pool.d/librenms.conf
$ vim /etc/php/8.1/fpm/pool.d/librenms.conf
```
```
; Start a new pool named 'www'.
; the variable $pool can be used in any directive and will be replaced by the
; pool name ('www' here)
[librenms]  <---- here!!! 
```

```
; Unix user/group of processes
; Note: The user is mandatory. If the group is not set, the default user's group
;       will be used.
user = librenms     <----  here!!!
group = librenms    <----  here!!!
```
```
; Note: This value is mandatory.
listen = /run/php-fpm-librenms.sock  <---- here!!!
```

## -Configure Web Server

```
$ vim /etc/nginx/conf.d/librenms.conf
```
```
server {
 listen      80;
 server_name 172.31.0.219;   <---- here!!!
 root        /opt/librenms/html;
 index       index.php;

 charset utf-8;
 gzip on;
 gzip_types text/css application/javascript text/javascript application/x-javascript image/svg+xml text/plain text/xsd text/xsl text/xml image/x-icon;
 location / {
  try_files $uri $uri/ /index.php?$query_string;
 }
 location ~ [^/]\.php(/|$) {
  fastcgi_pass unix:/run/php-fpm-librenms.sock;
  fastcgi_split_path_info ^(.+\.php)(/.+)$;
  include fastcgi.conf;
 }
 location ~ /\.(?!well-known).* {
  deny all;
 }
}
```
```
$ rm /etc/nginx/sites-enabled/default
$ systemctl restart nginx
$ systemctl restart php8.1-fpm
```

## -Enable lnms command completion
```
$ ln -s /opt/librenms/lnms /usr/bin/lnms
$ cp /opt/librenms/misc/lnms-completion.bash /etc/bash_completion.d/
```

## -Configure snmpd
```
$ cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf
```
```
$ vim /etc/snmp/snmpd.conf
```
```
# Change RANDOMSTRINGGOESHERE to your preferred SNMP community string
com2sec readonly  default         public    <---- here!!!

group MyROGroup v2c        readonly
view all    included  .1                               80
access MyROGroup ""      any       noauth    exact  all    none   none
```
```
$ curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
$ chmod +x /usr/bin/distro
$ systemctl enable snmpd
$ systemctl restart snmpd
```

## -Cron job
```
$ cp /opt/librenms/librenms.nonroot.cron /etc/cron.d/librenms
```
## -opy logrotate config
```
$ cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```
## -Web installer
```
$ chown librenms:librenms /opt/librenms/config.php
```
## -Final steps
login to http://172.31.0.219/
    
## -Troubleshooting
if you have a issues
```
$ sudo su - librenms
$ ./validate.php
```


# Integrations with Nagios plugins services

install monitoring-plugins
```
$ apt install monitoring-plugins
```
## -Setting up services
enable Services check
```
$ lnms config:set show_services true
```
check Services
```
$ lnms config:get show_services
``` 
### add discover services
```
$ vim /opt/librenms/config.php
```
Add the following line
```
# Enable the in-built services support (Nagios plugins)
#$config['show_services'] = 1;                                  
#$config['discover_services'] = true;                           <----  here!!!
#$config['discover_services_templates'] = true;                 <----  here!!!
```
### add nagios piugins
```
$ vim /opt/librenms/config.php
```
Add the following line
```
# Enable the in-built services support (Nagios plugins)
#$config['show_services'] = 1;                                  
#$config['discover_services'] = true;                           
#$config['discover_services_templates'] = true;                 
#$config['nagios_plugins']   = "/usr/lib/nagios/plugins";       <----  here!!!
```
```
$ chmod +x /usr/lib/nagios/plugins/*
```
## add services-wrapper.py to the current cron file
```
$ vim /etc/cron.d/librenms typically
```
```
*/5  *    * * *   librenms    /opt/librenms/check-services.php >> /dev/null 2>&1
*    *    * * *   librenms    cd /opt/librenms/ && php artisan schedule:run >> /dev/null 2>&1
*/5  *    * * *   librenms    /opt/librenms/services-wrapper.py 1   <---- here!!!

# Daily maintenance script. DO NOT DISABLE!
# If you want to modify updates:
```
![image](https://user-images.githubusercontent.com/117457958/211186804-b59cf588-a605-48c5-b4b4-8622fa992e83.png)

# Using LibreNMS

-login to http://172.31.0.219/ and click the Cylindrical symbol

![Screenshot 2023-01-05 224733](https://user-images.githubusercontent.com/117457958/211182411-de02ba77-3d2a-48f2-8da7-3945d46c2a94.png)

-Click -> Check Credentials and Input password

![Screenshot 2023-01-05 224823](https://user-images.githubusercontent.com/117457958/211183141-e4c6e9c1-5fd8-41f5-b0fb-baad76a1061e.png)

-Click -> Build Database

![Screenshot 2023-01-05 224843](https://user-images.githubusercontent.com/117457958/211182433-e0416a0e-acde-41e6-bd2f-2908ccb6f739.png)

-Click the key symbol and create Admin User

![Screenshot 2023-01-05 225039](https://user-images.githubusercontent.com/117457958/211182442-7e32c489-d627-4f27-87e0-10656267e017.png)

-Click the correct mark and click -> Finish Install

![Screenshot 2023-01-05 225102](https://user-images.githubusercontent.com/117457958/211182449-093d4a7e-844d-45ef-8e26-725a6f42a1a0.png)

-Click -> Validate Install

![Screenshot 2023-01-05 225122](https://user-images.githubusercontent.com/117457958/211182458-71941394-3b38-4179-bf7c-8417dbb33119.png)

```
mysql -u root
SET time_zone = '+7:00';
exit
```
```
systemctl restart mysql
```
-Click -> Attempt to automatically fix
![Screenshot 2023-01-05 225723](https://user-images.githubusercontent.com/117457958/211182464-c19e6ebb-bf35-4073-ac55-e31b64674e80.png)

## -Install SNMP and Add devices
```
$ sudo apt install snmpd
```
```
$ sudo nano /etc/snmp/snmpd.conf
```
- Change agentaddress
``` 
agentaddress udp:161,udp6:[::1]:161   <---- here!!!
#agentaddress 127.0.0.1,[::1]   <---- here!!!
```
- Add rocommunity
```
# Read-only access to everyone to the systemonly view
rocommunity librenmsv1 default   <---- here!!!
rocommunity  public default -V systemonly
rocommunity6 public default -V systemonly
```
```
$ sudo systemctl restart snmpd.service
```

-Add devices

![Screenshot 2023-01-05 225753](https://user-images.githubusercontent.com/117457958/211184073-407fd1a4-736b-4b41-9099-68a23e02847b.png)

-Input Hostname or Ip , Community and click -> Add Device

![image](https://user-images.githubusercontent.com/117457958/211184037-e78fe3e5-bf59-454e-b556-17f323c72fb3.png)
![image](https://user-images.githubusercontent.com/117457958/211184598-8521e88e-c03f-4fc9-8f99-0ba8cb59bfab.png)

-Click -> Devices -> All Devices -> Server 

![image](https://user-images.githubusercontent.com/117457958/211185275-01cff631-d1eb-4468-9f58-a40e644b764b.png)
![image](https://user-images.githubusercontent.com/117457958/211185324-3672799f-fbba-42dd-888d-a14b56312928.png)

# Reading the results of the tool
Overview 
 - System details
 - Processors
 - Overall Traffic
 - Memory
 - Storage
 - Recent Events
![image](https://user-images.githubusercontent.com/117457958/211185522-0f8f8099-410c-46ba-a2ae-bbd8fe9520cb.png)

Health
 - Processor (Tell the details of the processor hourly, daily, weekly, monthly.)
 วิธีอ่าน Graph Processor
 - แกน X คือค่าเวลาของ Graph
 - แกน Y คือค่า Processor ขณะที่ทำงานอยู่
สามารถบอกค่าเฉลี่ย ของ Processor ได้
![image](https://user-images.githubusercontent.com/117457958/211185957-58e0cb51-4bba-4cc8-baff-ec2584003d1b.png)
 - Memory (Able to tell RAM memory in different parts of the device, hourly, daily, weekly, monthly)
![image](https://user-images.githubusercontent.com/117457958/211186136-e7a55f75-98a9-4e96-a0ac-8e2ef6c8dc06.png)
 - Disk Usage (Able to tell memory in different parts of the device, hourly, daily, weekly, monthly)
![image](https://user-images.githubusercontent.com/117457958/211186220-a63ce453-9fcf-4dc6-b6bd-f7b595689c10.png)
 - Disk I/O(Able to tell the rate of entering-out of data in both hourly, daily, weekly, monthly)
![image](https://user-images.githubusercontent.com/117457958/211186276-ca41f98b-c8df-44de-92b1-d6a471139e08.png)


```
 - https://github.com/nantawatCharoenrat
 - https://github.com/paowick
 - https://github.com/Nattiya18
 - https://github.com/eiskawinnn
```









