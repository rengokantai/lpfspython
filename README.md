#### lpfspython
#####2 ssh
```
mkdir -p fsp/ssh_keys
cd fsp/ssh_keys
ssh-keygen -t rsa -b 2048
```
when prompting,type
```
./prod_key
```
then
```
cp prod_key.pub authorized_keys
```
create a vps on vultr without ssh key
connect.
```
apt-get update && apt-get upgrade && apt-get insatll -y fail2ban
```
config /etc/ssh/sshd_config
```
PasswordAuthentication no
UsePAM no
PermitRootLogin no
```
then add another user
```
groupadd deployers
mv /etc/sudoers /etc/sudoers.bak
chmod 0440 /etc/sudoers
```
add:
```
%deployers  ALL (ALL)=ALL ALL
```
```
useradd -c "ke" -m -g deployers deployer
passwd deployer
usermod -a -G deployers deployer
mkdir /home/deployer/.ssh
chown -R deployer:deployers /home/deployer/.ssh
scp prod_key.pub authorized_keys deployer@ip:~/.ssh
```
restart ssh:
```
service ssh reload
```
connect:
```
ssh -i ./prod_key deployer@...
```
#####cp3
```
apt-get update
apt-get install python-virtualenv python-dev git nginx
```
update ufw
```
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
```
enable virtualenv
```
virtualenv -p c:\python27\python.exe env\fspdeploy
```
Note, in windows the command should be
```
source /env/fsdeploy/Scripts/activate
```

deploy using ansible:with private keys and host file
```
#!/bin/bash
ansible-playbook deploy.yml --private-key=prod_key -K -u deployer -i hosts
```
#####chap4 server
add a A name to DNS provider, then create ke.conf under /etc/nginx/conf.d
```
upstream ke{
server localhost:8000 fail_timeout=0;
}

server{
listen 80;
server_name x.y.me;
rewrite ^(.*) https://server_name$1 permanent;
access_log /var/log/nginx/ke.access.log;
error_log /var/log/nginx/ke/error.log;
keepalive_timeout 5;

location /static{
autoindex on;
alias /home/deployer/ke/static;
}
location / {
proxy_set_header X-Forward-For $proxy_add_x_forward_for;
proxy_set_header Host $http_host;
proxy_redirect off;
if (!-f $request_filename){
proxy_pass http://ke
break;
}
}

location /socket.io{
proxy_pass http://ke/socket.io;
proxy_redirect off;
proxy_buffering off;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forward-For $proxy_add_x_forward_for;
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "Upgrade";
}

server{
server_name ...;
listen 443;
ssl_certificate /etc/nginx/ke/ke.crt;
ssl_certificate_key /etc/nginx/ke/ke.key;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
}
```

then
```
sudo rm /etc/nginx/sites-enabled/default
service nginx restart
```
######genssl
```
mkdir ssl_cert
cd ssl_cert
openssl genrsa -des3 -oout ssl.key 4096
openssl req -new -key ssl.key -out ssl.csr
```
recreate key the key file but without the pass phrase
```
openssl rsa -in ssl.key -out ke.key
openssl req -x509 -days 730 -in ssl.csr -signkey ke.key -out ke.crt
```
#####cp5 source control
create a deploy key. On production server:
```
ssh-keygen -t rsa -b 2048
```
name
```
./deploy_key
```
upload github, do not allow write access.Then install git
```
apt-get install git-core
```
using ssh-agent to clone repo:
```
ssh-agent bash -c 'ssh-add /..../deploy.key;git clone git@github.com:pj/git
```
update by pulling code:
```
ssh-agent bash -c 'ssh-add /..../deploy.key;git pull origin master
```
#####cp6 database
```
sudo apt-get install postgresql libpq-dev postgresql-client-common postgresql-client redis-server
```
setup
```
sudo su - postgres
createdb ke
createuser --superuser deployer
```
then
```
psql
ALTER ROLE deployer  WITH PASSWORD 'pass';
```
