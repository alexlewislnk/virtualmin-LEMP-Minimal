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
/tmp/install.sh --minimal --bundle LEMP
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

## Nginx and PHP Modifications
**PHP Versions and Modules**
```
apt -y install php8.0 php8.0-{bcmath,bz2,cgi,cli,common,curl,fpm,gd,igbinary,imagick,mbstring,memcached,mysql,opcache,readline,redis,xml,zip}
```
```
apt -y install php7.4 php7.4-{bcmath,bz2,cgi,cli,common,curl,fpm,gd,igbinary,imagick,mbstring,memcached,mysql,opcache,readline,redis,xml,zip} 
```
```
apt -y purge php5.6* php7.0* php7.1* php7.2* php7.3* php8.1*
```
```
phpenmod bcmath bz2 curl gd igbinary imagick mbstring memcached opcache readline redis xml zip
```

**/etc/nginx/nginx.conf**

This replacement nginx.conf file disables the fulle version info for Nginx, hardens the TLS/SSL settings, and enabled compression for certain file type to improve browser performance.
```
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.original
wget -O /etc/nginx/nginx.conf https://raw.githubusercontent.com/alexlewislnk/virtualmin-LEMP-Minimal/main/nginx.conf
```

**Setup default Nginx site to enforce strict SNI**
```
make-ssl-cert generate-default-snakeoil --force-overwrite
```
```
rm /etc/nginx/sites-enabled/default
primaryaddr=$(ip -f inet addr show dev "$defaultdev" | grep 'inet ' | awk '{print $2}' | cut -d"/" -f1 | cut -f1)
cat > /etc/nginx/sites-available/default << EOF
server {
        listen $primaryaddr:443 ssl default_server;
        listen $primaryaddr:80 default_server;
        server_name  _;
        error_log  /dev/null;
        access_log off;
        ssl_certificate     /etc/ssl/certs/ssl-cert-snakeoil.pem;
        ssl_certificate_key /etc/ssl/private/ssl-cert-snakeoil.key;
        ssl_stapling        off;
        ssl_ciphers         NULL;
        return 444;
}
EOF
ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/
```

**Restart Nginx**
```
systemctl restart nginx
```

## Virtualmin Post-Installation Wizard
From a web browser, log in to the Virtualmin console at port 10000, using the root user credentials, and complete the Post-Installation Wizard. For the initial setup, you should use the server's IP address in the URL instead of FQDN (https://x.x.x.x:10000)

During the setup wizard, you will be prompted for the MySQL root password created earlier in this guide. 

At the end of the Wizard, select the option to **Manage Enabled Features and Plugins**. These are the only features that should be enabled:
- Nginx website
- Nginx SSL website
- MySQL database
- Log file rotation

Next, select **System Settings** on the left menu, then click on **Re-check Configuration**.

## Final Tweaks

**Disable Unnecessary Services**

If you plan to host DNS elsewhere, disable Bind DNS
```
systemctl mask bind9
```

If you plan to host email elsewhere, disable Dovecot Mail Server
```
systemctl mask dovecot
```

Disable Proftp server (I strongly encourage the use of ssh-based sftp instead of ftp/ftps)
```
systemctl mask proftpd
```

**Harden Email Encryption**

Create Diffie-Hellman Key Pairs
```
openssl dhparam -out /etc/ssl/dhparam.pem 2048
```

**Create Initial Self-Signed Postfix Cert**
```
touch ~/.rnd
openssl req -new -x509 -nodes -out /etc/ssl/postfix.pem -keyout /etc/ssl/postfix.key -days 3650 -subj "/C=US/O=$HOSTNAME/OU=Email/CN=$HOSTNAME"
```

**Configure Email SSL/TLS**
```
postconf -e tls_medium_cipherlist=ECDH+AESGCM+AES128:ECDH+AESGCM:ECDH+CHACHA20:ECDH+AES128:ECDH+AES:DHE+AES128:DHE+AES:RSA+AESGCM+AES128:RSA+AESGCM:\!aNULL:\!SHA1:\!DSS
postconf -e tls_preempt_cipherlist=yes
postconf -e smtpd_use_tls=yes
postconf -e smtpd_tls_loglevel=1
postconf -e smtpd_tls_security_level=may
postconf -e smtpd_tls_auth_only=yes
postconf -e smtpd_tls_protocols=\!SSLv2,\!SSLv3,\!TLSv1,\!TLSv1.1
postconf -e smtpd_tls_ciphers=medium
postconf -e smtpd_tls_mandatory_protocols=\!SSLv2,\!SSLv3,\!TLSv1,\!TLSv1.1
postconf -e smtpd_tls_mandatory_ciphers=medium
postconf -e smtpd_tls_cert_file=/etc/ssl/postfix.pem
postconf -e smtpd_tls_key_file=/etc/ssl/postfix.key
postconf -e smtpd_tls_dh1024_param_file=/etc/ssl/dhparam.pem
postconf -e smtp_use_tls=yes
postconf -e smtp_tls_loglevel=1
postconf -e smtp_tls_security_level=may
postconf -e smtp_tls_protocols=\!SSLv2,\!SSLv3,\!TLSv1,\!TLSv1.1
postconf -e smtp_tls_ciphers=medium
postconf -e smtp_tls_mandatory_protocols=\!SSLv2,\!SSLv3,\!TLSv1,\!TLSv1.1
postconf -e smtp_tls_mandatory_ciphers=medium
postconf -e smtp_tls_cert_file=/etc/ssl/postfix.pem
postconf -e smtp_tls_key_file=/etc/ssl/postfix.key
systemctl restart postfix
```

**Restrict Mail protocols**

Since we are not using the Virtualmin’s mail services, then let’s lock down the Postfix SMTP server so it cannot be an attack target. We cannot disable it completely as it will be needed to send outbound email from your server. We configure it so connections are only accepted from the server itself.
```
postconf -e inet_interfaces=127.0.0.1
systemctl restart postfix
```

**Finished – Reboot**
```
reboot
```
