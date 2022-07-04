# Pomocné skripty pro předmět KIV/SPOS, autor: Jaroslav Lehečka
## Základní instalace

```bash
apt-get install vim
apt-get install mc
apt-get install curl
apt-get install iptables
systemctl enable ssh
```

```bash
apt-get install bind9 nginx mdadm lvm2 libapache2-mod-php7.4 fail2ban bind9 dnsutils nginx mariadb-server
```
## Obnova rozbořeného disku
- Recovery mode
- mount -t ext4 -o rw,remount /dev/sda1 /
- blkid
- echo "UUID=52e062e0-716c-4828-9bf1-05b93fdaef93 / ext4 errors=remount-ro 0 1" > /etc/fstab
- Následně budu schopen rebootu
- zdroj:
- https://askubuntu.com/questions/435965/accidentally-deleted-etc-fstab-file


## Síťování
```bash
ip addr add 192.168.1.50/24 dev enp0s3
ip addr del 192.168.1.50/24 dev enp0s3
ip r

ip route add 192.168.1.0/24 dev enp0s3
ip route add default via 192.168.1.1

více na:
https://access.redhat.com/sites/default/files/attachments/rh_ip_command_cheatsheet_1214_jcs_print.pdf
```

## Zabezpečení operačního systému Linux

```bash
apt-get install fail2ban


FAIL2BAN !!!!!

mkdir ~/.ssh
curl https://gitlab.com/leheckaj.keys >> ~/.ssh/authorized_keys

echo "PermitRootLogin yes
StrictModes yes
RSAAuthentication yes
PubkeyAuthentication yes
PasswordAuthentication no" >> /etc/ssh/sshd_config

ssh -vvvv root@192.168.0.222


iptables -A INPUT -s  192.168.0.0/24  -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -s  147.228.0.0/16  -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 22  -j DROP

iptables-save > /etc/network/iptables
echo "post-up /sbin/iptables-restore /etc/network/iptables" > /etc/network/interfaces

```

## RAID
```bash
apt install mdadm

mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sd[bcd]
mdadm --stop /dev/md0
mdadm --remove /dev/md0

Výměna vadného disku /dev/sda1 za /dev/sdc1:
mdadm --fail /dev/md0 /dev/sda1
mdadm --remove /dev/md0 /dev/sda1
mdadm --add /dev/md0 /dev/sdc1

Zkráceně to samé:
mdadm /dev/md0 --fail /dev/sda1 --remove /dev/sda1 --add /dev/sdc1


mdadm --zero-superblock /dev/sda
cat /proc/mdstat
```


## LVM
```bash
apt install lvm2
--------------------TVORBA---------------------------
pvcreate /dev/vdb
pvcreate /dev/vdc1
pvcreate /dev/vdc2
pvcreate /dev/vdc3
    Případně: pvcreate /dev/sd{b,c,d}

vgcreate data /dev/vdb /dev/vdc1 /dev/vdc2 /dev/vdc3
    Případně: vgcreate data /dev/sd{b,c,d}
lvcreate -n db -L 1.5G data
lvcreate -n share -L 200M data

Kontrola:
cat /proc/mdstat

mkfs.ext4 /dev/data/db
nebo:
mkfs.xfs /dev/data/db
mount /dev/data/db /mnt/

-----------------------ZNIČENÍ-----------------------
umount /mnt
lvremove /dev/data/db
vgremove data
pvremove data /dev/vdb /dev/vdc1 /dev/vdc2 /dev/vdc3
    Případně: pvremove data /dev/sd{b,c,d}

--------------VYMENA DISKU-----------------------
Zdroj: http://www.microhowto.info/howto/replace_one_of_the_physical_volumes_in_an_lvm_volume_group.html
Nový disk: /dev/sdc, starý disk: /dev/sdb
-------------------------------------------------------
pvcreate /dev/sdc
vgextend data /dev/sdc
pvmove /dev/sdb /dev/sdc
vgreduce data /dev/sdb
pvremove /dev/sdb
```

## Zrychlená tvorba RAID LVM
```bash

mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sd[bcd]
pvcreate /dev/md0
vgcreate data /dev/md0
lvcreate -n db -L 4M data
mkfs.ext4 /dev/data/db

e2label /dev/data/db db-label

pro XFS:
xfs_admin -L "db-label" /dev/sdb1 

mkdir /opt/share
echo "LABEL=db-label     /opt/share    ext4   defaults 0 0"  >> /etc/fstab
mount -a

touch /opt/share/soubor{0..100}
```

## Zrychlené zničení RAID LVM
```bash
cd /opt/share
rm *
cd - #(návrat do složky, kde jsem byl předtím)
umount /opt/share

# Zakomentování automatického připojování při startu
awk '$1 == "LABEL=db-label"  {$1 = "#LABEL=db-label"} {print} ' test.txt > out.txt
cat out.txt > /etc/fstab
rm out.txt

lvremove /dev/data/db
vgremove data
pvremove data /dev/sd{b,c,d}

mdadm --stop /dev/md0
mdadm --remove /dev/md0
```

## Zrychlené nastavení DNS
apt-get install bind9 dnsutils

```bash
domain=test.spos
echo "\$TTL    604800
@   IN  SOA $domain. root.localhost. (

                  2     ; Serial / YYYYMMDDXX
             604800     ; Refresh / seconds
              86400     ; Retry / seconds
            2419200 ; Expire / seconds
             604800 )   ; Negative Cache TTL / explicitni TTL

@         IN      NS                ns
ns        IN      A                 127.0.0.1
mail      IN      CNAME             ns
www       IN      CNAME             ns
posta     IN      CNAME             ns
@         IN      MX           10   mail
@         IN      MX           20   posta
txt       IN      TXT               \"ahoj svete\"" > /etc/bind/db.$domain


echo "
zone \"$domain.\" in {
    type master;
    file \"/etc/bind/db.$domain\";
};" >> /etc/bind/named.conf.local

Pro jistotu přidej ručně:
sed '3 i         listen-on { 192.168.0.224; };' /etc/bind/named.conf.options

service bind9 restart

host test.spos 192.168.0.224
```

## Apache2 práce pro zákaz na rozjetí na vícero portech
```bash
apt-get install libapache2-mod-php7.4
Instalace php + apache

awk '$2 == "*:80>"  {$2 = "*:8080>"} {print} ' /etc/apache2/sites-available/000-default.conf  > /etc/apache2/sites-available/port1.conf
awk '$2 == "*:80>"  {$2 = "*:8181>"} {print} ' /etc/apache2/sites-available/000-default.conf  > /etc/apache2/sites-available/port2.conf

a2dissite 000-default
a2ensite port1
a2ensite port2

systemctl reaload apache2
rm /etc/apache2/sites-available/000-default.conf

Zákaz služeb:
/etc/php/7.4/apache2/php.ini
v disable_functions = phpinfo

echo " <?php echo date('Y-m-d H:i:s'); phpinfo(); ?>" > /var/www/html/index.php
rm /var/www/html/index.html


Manuálně uprav porty v /etc/apache2/ports.conf
Listen 8181
Listen 8080
```

## Apache2 rozjetí šifrované části
```bash
Do /etc/apache2/sites-available/... přidám:

	<Location /secret>
                AuthType Basic
                AuthName "Restricted Access"
                AuthUserFile /etc/apache2/.htpasswd
                Require user pepa
        </Location>

mkdir /var/www/html/secret
htpasswd -c /etc/apache2/.htpasswd pepa
echo "tst" > /var/www/html/secret/tajne.txt
curl http://pepa:123@127.0.0.1:80/secret/tajne.txt


Zabezpečení PHPMyAdmin zde:
https://www.linode.com/docs/guides/how-to-secure-phpmyadmin/   ----> Setting Up Password Based Authentication
http://192.168.0.250/phpmyadmin-jwef82r68662b/index.php
pepa/pepa
```

## Nginx HA
```bash
apt-get install nginx

Zkontroluj:

echo "
upstream backend  {
        server 127.0.0.1:8080;
        server 127.0.0.1:8181;
}

server {
        listen 80;
        server_name www.test.spos;

        return 301 https://192.168.0.224;
}

server {
        listen 443 ssl default_server;
        server_name www.test.spos;

	include snippets/snakeoil.conf;
	
        location / {
                proxy_pass http://backend;
                include proxy.include;
        }
}" > /etc/nginx/sites-enabled/lb.conf

echo "proxy_set_header   Host \$http_host;
proxy_set_header   X-Real-IP \$remote_addr;
proxy_set_header   X-Forwarded-For \$proxy_add_x_forwarded_for;
proxy_set_header   X-Forwarded-Proto \$scheme;
proxy_connect_timeout      90;
proxy_send_timeout         300;
proxy_read_timeout         300;" > /etc/nginx/proxy.include

service nginx reload
service nginx restart
```

## MySQL
```bash
apt-get install mariadb-server

service mysql start
mysql -uroot

show databases;
create database db01;
use db01;
CREATE TABLE table01(
   id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY,
   firstname VARCHAR(30) NOT NULL,
   lastname VARCHAR(30) NOT NULL,
   email VARCHAR(50),
   reg_date TIMESTAMP
);

// Přístup odkudkoliv
CREATE USER 'spos-test'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON db01.* to 'spos-test'@'%' IDENTIFIED BY 'password';
flush privileges;

CREATE USER 'spos-ro'@'%' IDENTIFIED BY 'password';
GRANT SELECT ON db01.* to 'spos-ro'@'%' IDENTIFIED BY 'password';
flush privileges;

mysql -uspos-test -ppassword
use db01;
INSERT INTO table01 (firstname, lastname, email, reg_date) VALUES ("Jarda", "Lehecka", "test@test.spos", NOW());

mysql -uspos-ro -ppassword
use db01;
INSERT INTO table01 (firstname, lastname, email, reg_date) VALUES ("Jarda", "Lehecka", "test@test.spos", NOW());
=====> NEPŮJDE


/etc/mysql/mariadb.conf.d/50-server.cnf
        bind-adress: 0.0.0.0
	
service mysql restart

apt-get install php-mysqlnd



mysqldump db01 | gzip > db01.sql.gz
mysql >>>>>>>>>>>> delete from table01;
gunzip -c db01.sql.gz | mysql -udb01 -ppassword db01

```


```bash
<?php
$servername = "localhost";
$username = "spos-ro";
$password = "password";
$dbname = "db01";

// Create connection
$conn = new mysqli($servername, $username, $password, $dbname);
// Check connection
if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
} 

$sql = "SELECT id, firstname, lastname FROM MyGuests";
$result = $conn->query($sql);

if ($result->num_rows > 0) {
    // output data of each row
    while($row = $result->fetch_assoc()) {
        echo "id: " . $row["id"]. " - Name: " . $row["firstname"]. " " . $row["lastname"]. "<br>";
    }
} else {
    echo "0 results";
}
$conn->close();
?>
```

## Postgres

```bash
apt-get install postgresql php-pgsql
apt --purge remove postgresql-13 -y

psql db01
\c db01

\d table01
\dt
-----------------------------------
psql
CREATE USER db01 WITH PASSWORD 'pasword';
CREATE DATABASE db01 WITH OWNER=db01;
\c db01
CREATE TABLE table01(
   id SERIAL PRIMARY KEY,
   firstname VARCHAR(30) NOT NULL,
   lastname VARCHAR(30) NOT NULL,
   email VARCHAR(50),
   reg_date TIMESTAMP
);
GRANT ALL PRIVILEGES ON TABLE table01 to db01;
\q
------------------------------------
psql postgresql://db01:pasword@127.0.0.1 db01
INSERT INTO table01(firstname, lastname, email, reg_date) values ('Jarda', 'Lehecka', 'mail', now());
SELECT * FROM table01;
\q
------------------------------
psql
\c db01
DROP TABLE table01;
\q
psql
DROP database db01;
DROP user db01;
\q
```


## Hromadné generování

```bash
INSERT INTO table01(firstname, lastname, email, reg_date) values ('Jarda', 'Lehecka', 'mail', now());

for i in `seq 1 2`; do 
   echo "INSERT INTO table01(firstname, lastname, email, reg_date) values ('Jarda', 'Lehecka', 'mail@mail.spos', now());" | psql db01
done

for i in `seq 1 50`; do 
   echo "INSERT INTO table01(firstname, lastname, email, reg_date) values ('Jarda', 'Lehecka', 'mail@mail.spos', now());" | mysql db01
done
```

## Postfix

```bash
apt-get install postfix
apt-get install mailutils
apt-get install mutt


awk '$1 == "mynetworks"  {$1 = "#mynetworks"} {print} ' /etc/postfix/main.cf  > /etc/postfix/main.cf.edit
awk '$1 == "myhostname"  {$1 = "#myhostname"} {print} ' /etc/postfix/main.cf.edit  > /etc/postfix/main.cf.edit2

echo "myhostname=mail.$domain" >> /etc/postfix/main.cf.edit2
echo "mynetworks=192.168.0.0/24" >> /etc/postfix/main.cf.edit2
echo "mydomain=$domain" >> /etc/postfix/main.cf.edit2
echo "home_mailbox=Maildir/" >> /etc/postfix/main.cf.edit2
echo "mailbox_command =" >> /etc/postfix/main.cf.edit2

rm /etc/postfix/main.cf.edit
mv /etc/postfix/main.cf.edit2 /etc/postfix/main.cf


service postfix restart
mutt -f ~/Maildir
```

## Dovecot IMAP

```bash
apt-get install dovecot-imapd


echo "listen = *, ::" >> /etc/dovecot/dovecot.conf

V /etc/dovecot/conf.d/10-mail.conf uprav mail_location
mail_location:~/Maildir

Sekci auth v /etc/dovecot/conf.d/10-master.conf  pozměň na:
# Postfix smtp-auth
unix_listener /var/spool/postfix/private/auth {
  mode = 0666
  user = postfix
  group = postfix
 }
```

```bash
Z roota:
echo "Test" | mail -s "Testovaci mail" pepa@test.spos
su - pepa
mutt -f ~/Maildir/
```

```bash
telnet localhost 143
# https://tools.ietf.org/html/rfc3501
1 LOGIN pepa pepa
2 LIST "" "*"
3 STATUS INBOX (MESSAGES)
4 EXAMINE INBOX
5 FETCH 1 BODY[]
6 LOGOUT
```

## NFS
```bash
apt-get install nfs-server

mkdir /srv/share{1,2,3}
vim /etc/exports

/srv/share1       10.228.67.0/24(rw) 147.228.67.0/24(ro)
/srv/share2       10.228.67.0/24(rw,no_root_squash)
/srv/share3       10.228.67.0/24(rw,no_subtree_check,
                            all_squash,anonuid=6000,anongid=6000)

chmod 777 /srv/share1

ip a a 10.228.67.50/24 dev ens18

mkdir /mnt/share{1,2,3}
mount -t nfs  10.228.67.50:/srv/share1 /mnt/share1

touch test.txt
ls -al
```
## Samba
```bash
apt-get install samba smbclient cifs-utils -y

echo "
[share1]
        comment = Prvni share co jsem kdy vyrobili
        path = /srv/test
        browsable = yes        
	writable = yes
        guest ok = yes
        create mask = 0600
        valid users = jindra
        directory mask = 0700" >> /etc/samba/smb.conf

adduser jindra
smbpasswd -a jindra
addgroup --gid 6001 cifs
usermod -G cifs jindra

mount -t cifs //localhost/share1 /mnt/test -o username=jindra

/etc/samba/smb.conf:

```
