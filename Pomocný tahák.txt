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
apt-get install bind nginx mdadm lvm2 libapache2-mod-php7.4 fail2ban
```

## Zabezpečení operačního systému Linux

```bash
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
