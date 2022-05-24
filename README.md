# Pomocné skripty pro předmět KIV/SPOS, autor: Jaroslav Lehečka
## Zabezpečení operačního systému Linux

mkdir ~/.ssh
curl https://gitlab.com/leheckaj.keys >> ~/.ssh/authorized_keys

echo "PermitRootLogin yes
StrictModes yes
RSAAuthentication yes
PubkeyAuthentication yes
PasswordAuthentication no" >> /etc/ssh/sshd_config
