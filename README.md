# Virtualmin's Minimal LEMP Server
General instructions for install and setup. The minimal installation excludes the full mail processing stack (IMAP/POP servers, SpamAssassin and ClamAV).

This assumes you have already completed the basic setup of your Ubuntu 20.04 LTS Server. 
I suggest checking out my scripts for [Ubuntu Setup](https://github.com/alexlewislnk/Ubuntu-Setup) and [Emerging Threats Firewall](https://github.com/alexlewislnk/ET-Firewalld).

## Install Virtualmin

**Add repository for latest version of Nginx**
```
add-apt-repository ppa:ondrej/nginx-mainline && apt update -y
```

**Add repository for latest version of PHP**
```
add-apt-repository ppa:ondrej/php && apt update -y
```

**Download the Virtualmin installation script**
```
cd /tmp
wget http://software.virtualmin.com/gpl/scripts/install.sh
chmod +rx /tmp/install.sh
```

**Run the install script**
```
/tmp/install.sh -–minimal --bundle LEMP
```

**Configure Fail2Ban to work with the Firewall**
```
virtualmin config-system --include Fail2banFirewalld
```

**Make sure Fail2Ban is enabled and running**
```
systemctl enable fail2ban ; systemctl restart fail2ban
```

## Move MySQL Data Directory
I like to locate the MySQL data under the /home directory so everything for Virtualmin is under a single path. This can be helpful if you later need to add disk space my moving /home to a separate drive.

**Stop the MySQL service**
```
service mysql stop
```

**Create new data directory**
```
mkdir /home/mysql
chown mysql:mysql /home/mysql
```

**Define new data directory in MySQL config file**
```
cp /etc/mysql/mysql.conf.d/mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf.save
```
```
sed -i '/datadir/d' /etc/mysql/mysql.conf.d/mysqld.cnf
```
```
echo "datadir = /home/mysql" >> /etc/mysql/mysql.conf.d/mysqld.cnf
```

**Update AppArmor**
```
echo "alias /var/lib/mysql/ -> /home/mysql/," >> /etc/apparmor.d/tunables/alias
service apparmor restart
```

**Initialize New Data Directory** *(don’t worry, we will add a password during the Virtualmin setup later)*
```
mysqld --initialize-insecure
```

**Start MySQL**
```
service mysql start
```

**Create SQL System Maintenance User** *(This will include a new longer random password for the maintenance user than normally provided by Ubuntu's setup)*
```
RANDOM1=`< /dev/urandom tr -dc '[:alnum:]' | head -c${1:-64}`
cp /etc/mysql/debian.cnf /etc/mysql/debian.cnf.bak
cat > /etc/mysql/debian.cnf <<EOF
# Automatically generated for Debian scripts. DO NOT TOUCH!
[client]
host     = localhost
user     = debian-sys-maint
password = $RANDOM1
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = debian-sys-maint
password = $RANDOM1
socket   = /var/run/mysqld/mysqld.sock
basedir  = /usr
EOF
echo "CREATE USER 'debian-sys-maint'@'localhost' IDENTIFIED BY '$RANDOM1';" | mysql -u root
echo "GRANT ALL PRIVILEGES ON *.* TO 'debian-sys-maint'@'localhost';" | mysql -u root
echo "GRANT PROXY ON ''@'' TO 'debian-sys-maint'@'localhost' WITH GRANT OPTION;" | mysql -u root
```

**Create Random Password for MySQL root user**
```
RANDOM1=`< /dev/urandom tr -dc '[:alnum:]' | head -c${1:-32}`
echo "ALTER USER 'root'@'localhost' IDENTIFIED BY '$RANDOM1';" | mysql -u root
echo "
MySQL root password has been set to $RANDOM1
Please record password for use later in this setup 
and save in your password manager."
```

##Nginx Modifications

(more to come)

## Virtualmin Post-Installation Wizard
From a web browser, log in to the Virtualmin console at port 10000, using the root user credentials, and complete the Post-Installation Wizard. For the initial setup, you should use the server's IP address in the URL instead of FQDN (https://x.x.x.x:10000)

During the setup wizard, you will be prompted for the MySQL root password created earlier in this guide. 

At the end of the Wizard, select the option to **Manage Enabled Features and Plugins**. These are the only feature that should be enabled:
- Nginx website
- SSL website
- Log file rotation
- MySQL database
- Protected web directories





