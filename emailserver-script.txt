#!/bin/bash
# Email Server Script Created by Salman A. Francis Version 1
# This Script is tested on CentOs 7 
#Disable SeLinux
SELinuxConfig=/etc/selinux/config
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' $SELinuxConfig
setenforce 0
# check if postfix is available if not install it
Service1=postfix
Service2=dovecot
configfile=/etc/postfix/main.cf
Status=$(systemctl is-active $Service1)
doveconfig=/etc/dovecot/dovecot.conf
doveconfdir=/etc/dovecot/conf.d/
if [ "${Status}" = "active" ]; then
echo "${Service1}" is installed and running
else
echo "Postfix will be installed now"
yum -y install postfix
systemctl restart postfix
systemctl enable postfix
fi
# check if dovecot is installed in the system if not install it
Status=$(systemctl is-active $Service2)
if [ "${Status}" = "active" ]; then
echo "${Service2}" is installed and running
else
echo "Dovecot will be installed now"
yum -y install dovecot
systemctl restart dovecot
systemctl enable dovecot
fi
#Installing & Enabling Firewall Rules for Email
yum -y install firewalld
systemctl restart firewalld  && systemctl enable firewalld
systemctl mask iptables
firewall-cmd --add-port=25/tcp --permanent
firewall-cmd --add-port=110/tcp --permanent
firewall-cmd --add-port=465/tcp --permanent
firewall-cmd --add-port=143/tcp --permanent
firewall-cmd --add-port=995/tcp --permanent
firewall-cmd --add-port=993/tcp --permanent
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --add-port=443/tcp --permanent
firewall-cmd --add-port=587/tcp --permanent
firewall-cmd --reload
# Backing up main config file of postfix before making changes
cp /etc/postfix/main.cf /etc/postfix/main.cf.org
# Getting Domain info
echo "Please Enter Hostname e.g Dragon.domain.com, mail.domain.com etc "
read hostname
echo "You have entered $hostname " is this correct ? Y/N
read opt
if [[ $opt =~ ^[yY]$ ]]; then

#changing hostname in postfix config file
sed -i 's/#myhostname = host.domain.tld/myhostname = '$hostname'/g' $configfile
else
echo "Please Restart the Script"
exit 1
fi

echo "Please Enter The Name Of Your Domain e.g tekco.net, itpings.com "
read domainname
echo "You entered $domainname " is this correct ? Y/N 
read options
if [[ $options =~ ^[yY]$ ]]; then
sed -i 's/#mydomain = domain.tld/mydomain = '$domainname'/g' $configfile
sed -i 's/#myorigin = $mydomain/myorigin = $mydomain/g' $configfile
sed -i 's/#inet_interfaces = all/inet_interfaces = all/g' $configfile
sed -i 's/inet_interfaces = localhost/#inet_interfaces = localhost/g' $configfile
sed -i 's/mydestination = $myhostname, localhost.$mydomain, localhost/mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain/g' $configfile
else
echo "Please Restart the Script"
exit 1
fi
echo "Please Enter Your Network e.g 10.1.1.0"
read network
echo "Please Enter Subnetmask e.g 8, 16, 24, 32"
read subnet
echo "You entered $network & Subnetmask is $subnet" is this correct ? Y/N
read net
if [[ $net =~ ^[yY]$ ]]; then
sed -i -e 's/#mynetworks = 168.100.189.0\/28, 127.0.0.0\/8/mynetworks = 127.0.0.0\/8,'$network'\/'$subnet'/g' $configfile
sed -i 's/#home_mailbox = Maildir\//home_mailbox = Maildir\//g' $configfile
sed -i 's/#smtpd_banner = $myhostname ESMTP $mail_name/smtpd_banner = $myhostname ESMTP/g' $configfile
sed -i '/^{#smtpd_banner = $myhostname ESMTP ($mail_version)}$/d' $configfile
else
echo "Please Restart the Script"
exit 1
fi
echo "# for SMTP-Auth
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = \$myhostname
smtpd_recipient_restrictions = permit_mynetworks, permit_auth_destination, permit_sasl_authenticated, reject" >> $configfile
systemctl restart postfix
#Setting up DoveCot
cp $doveconfig $doveconfig\.org
sed -i 's/#protocols = imap pop3 lmtp/protocols = imap pop3 lmtp/g' $doveconfig
sed -i 's/#listen = \*, ::/listen = \*/g' $doveconfig
systemctl restart dovecot
#Bakcup Dovecot Authfile
cp $doveconfdir/10-auth.conf $doveconfdir/10-auth.conf.org
sed -i 's/#disable_plaintext_auth = yes/disable_plaintext_auth = no/g' $doveconfdir/10-auth.conf
sed -i 's/auth_mechanisms = plain/auth_mechanisms = plain login /g' $doveconfdir/10-auth.conf
sed -i -e 's/#mail_location = /mail_location = maildir:~\/Maildir /g' $doveconfdir/10-mail.conf
# uncomment line 96-98 in $doveconfdir/10-master.conf and add some text beneath it
sed -i '96,98 s/^\ \ #//g' $doveconfdir/10-master.conf
sed -i '97 i user = postfix\n group = postfix\n' $doveconfdir/10-master.conf
systemctl restart dovecot
#Installing Rainloop
#!/bin/bash
# Installing Remi Repo
yum -y install https://rpms.remirepo.net/enterprise/remi-release-7.rpm
yum -y install yum-utils
yum-config-manager --enable remi-php74
yum -y update
yum -y install php php-cli php-common php-mbstring php php-gd php-intl php-pecl-apcu php-opcache php-json php-pecl-zip php-pear php-pecl-imagick php-fpm php-pecl-redis5 php-pgsql php-common php-pdo php-xml php-lz4 php-pecl-crypto php-pecl-rar php-pecl-pq php-pecl-lzf php-cli php-pecl-apcu-bc
# Installing Epel Rep
yum -y install epel-release
# Installing Apache Web Server
yum -y install httpd unzip wget
systemctl restart httpd
systemctl enable httpd
# Rainloop Setup
cd /opt
wget http://repository.rainloop.net/v2/webmail/rainloop-latest.zip
mkdir /var/www/html/rainloop
unzip rainloop-latest.zip -d /var/www/html/rainloop
find /var/www/html/rainloop -type d -exec chmod 755 {} \;
find /var/www/html/rainloop -type f -exec chmod 644 {} \;
chown -R apache:apache /var/www/
cd /etc/httpd/conf
sed -i 's/DocumentRoot\ \"\/var\/www\/html"/DocumentRoot\ \"\/var\/www\/html\/rainloop"/g' httpd.conf
systemctl restart httpd
