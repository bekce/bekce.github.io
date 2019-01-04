---
layout:     post
title:      Securing MongoDB with TLS, Authentication and LetsEncrypt
date:       2019-01-03 11:20:00
categories: sysadmin
---

This is a guide to build a dedicated MongoDB server on a public or private network to serve for your PaaS, with valid TLS certificates and authentication enabled to guard against outsiders.

I've checked some major MongoDB as a service (a.k.a. 'cloud', if you like fancy words) providers and one called mLab does NOT even support TLS on database connections with their affordable package (not free) which is clearly a joke to me. Many other providers don't even mention TLS/SSL on their features list at all so I guess they simply assume every potential use case involves same-datacenter private networking, or some VPN or SSH tunnelling magic, blah. Or, it could be the case that they simply hadn't upgraded to MongoDB 4.0 which improved TLS connection support a lot.

Note that in this guide we won't mention anything about firewall setups as it's usually vendor specific.
You should basically allow ports 22, 80 and 27017 for this setup to work correctly.
You can also disable default firewall in CentOS servers by `systemctl disable firewalld`, it's your call.

1\. I usually pick CentOS for production stuff, as Ubuntu is too flakey and wobbly (a.k.a. you leave a server running for a couple of years and *BAM!*, they change the *init* system!). Run everything with `root`, no bullshitting around here.

2\. MongoDB might require huge burst memory from time to time, and since you're reading this I assume you've got a budget friendly server with low memory so better setup a swapfile first. Hopefully your server has an SSD though! I've been using [Vultr.com](https://www.vultr.com/?ref=6935650) for more than 2 years now and they're extremely good on performance/price level).

```sh
# check whether you have swap on
swapon -s
# if not, go on
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
# check again
swapon -s
free -m
# make permanent
echo '/swapfile   swap    swap    sw  0   0' >> /etc/fstab
```

3\. Install MongoDB 4.0. You are encouraged to only use 4+ versions because of the improved TLS security and the use of native libraries. Any newer version (4.2, 4.4, etc) would be fine as long as you use the latest yum repository.

```sh
cat>/etc/yum.repos.d/mongodb-org-4.0.repo <<\EOF
[mongodb-org-4.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc
EOF

yum install -y mongodb-org libcurl openssl
service mongod start
systemctl enable mongod
```

Run in `mongo` shell and add your first user like below.

Read [this](https://docs.mongodb.com/manual/tutorial/enable-authentication/) to learn about adding more users and roles.
It's recommended to NOT use admin user for projects, and rather setup a new user account for every application/project you want to develop and only give permissions to a specific set of databases.
You can have a single mongodb server to cater for multiple applications/projects without compromising security (assuming mongod does not have a flaw in their implementation).

```
use admin
db.createUser(
  {
    user: "admin",
    pwd: "password",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
  }
)
```

4\. Install fail2ban. This will protect your server against some types of attacks.

```sh
yum install epel-release
yum install fail2ban

cat>/etc/fail2ban/jail.local <<\EOF
[DEFAULT]
# Ban hosts for one hour:
bantime = 3600

# Override /etc/fail2ban/jail.d/00-firewalld.conf:
banaction = iptables-multiport

[sshd]
enabled = true

[mongo-auth]

enabled = true
filter  = mongo-auth
logpath = /var/log/mongodb/mongod.log
maxretry = 3
port    = 27017
banaction = iptables-multiport[name="mongo", port="27017"]

bantime = 86400
findtime = 300
EOF

systemctl enable fail2ban
service fail2ban restart
```

I've also activated mongo-auth plugin so it will ban clients that fails too many times to authenticate on mongod.
You can (and should) test this by trying to login from another machine (after setting up everything until end first) and watch fail2ban via `fail2ban-client status mongo-auth`.

5\. Set up letsencrypt. First make sure `hostname` actually produces the correct hostname for your server. And that hostname resolves to the public ip of your server.

```sh
yum -y install yum-utils certbot
certbot certonly --standalone -d my.example.com

cat>/root/renew.sh <<\EOF
#!/bin/bash
HOSTNAME=`hostname`
cat /etc/letsencrypt/live/$HOSTNAME/fullchain.pem /etc/letsencrypt/live/$HOSTNAME/privkey.pem > /etc/ssl/mongodb.pem
chmod 644 /etc/ssl/mongodb.pem
service mongod restart
EOF
chmod +x /root/renew.sh
```

Insert this your `crontab -e`:
```sh
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin

0 0,12 * * * certbot renew --renew-hook /root/renew.sh > /root/certbot-cron.log 2>&1
```

Run renew.sh once, this will put the generated key file in the right place.

    /root/renew.sh

The only 'gotcha' of this solution is that the server has to be restarted about every 2 months (the renew period for letsencrypt certificates), which will momentarily break connections. Alas, most client implementations can handle it gracefully by auto-retrying in a second, and with the help of oplog, you are unlikely to lose any data, but if your workload cannot tolerate that, use replica sets by setting up another server.

6\. Tweak mongodb to accept ssl and authorization

Add/change following section in `/etc/mongod.conf`:

```sh
# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0 # enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.
  ssl:
    mode: requireSSL
    PEMKeyFile: /etc/ssl/mongodb.pem

security:
  authorization: enabled
```

Restart service: `service mongod restart`

7\. Test

Now SSL and authentication is activated on your server and it is ready to go. 
Try connecting to your server via: `mongo --ssl -u admin -p password --authenticationDatabase "admin" my.example.com`.

For other applications, you need to use a mongodb uri like this: `mongodb://admin:password@my.example.com:27017/db_name?ssl=true`, and supply `admin` as your authenticationDatabase setting seperately.

PS: I've compiled this guide from the history of commands I've performed on a machine and I may have missed some commands in between.
If there's any issue/error running these in order, please comment below, thanks.

Happy new 2019!
