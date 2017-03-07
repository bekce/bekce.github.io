---
layout:     post
title:      "Sieve Email Filtering with Dovecot and Roundcube"
date:       2016-07-09
categories: sysadmin
---

Sieve filtering is a very useful feature on your own email server as it lets you to define powerful filtering options for your email server. In this guide I will explain the steps I used to enable sieve filtering on my server.

I assume your (optionally multi-domain, debian based) email server is already configured with postfix, dovecot and roundcube and you would like to enable sieve filtering and manage those filters using roundcube’s managesieve plugin.

1.  Install packages: `apt-get install dovecot-sieve dovecot-managesieved`

2.  Edit `/etc/dovecot/conf.d/90-sieve.conf`

        plugin {
          sieve = /var/vmail/%d/%n/sieve/dovecot.sieve
          sieve_dir = /var/vmail/%d/%n/sieve
        }

    `sieve` and `sieve_dir` variables are user specific. Here `%d` and `%n` are variables, explanation is in `/usr/share/doc/dovecot-core/dovecot/wiki/Variables.txt.gz`. You need to set these variables in accordance with your (multi-domain) installation.  

3.  Edit `/etc/dovecot/conf.d/20-managesieve.conf`, enable services below by uncommenting them.

        service managesieve-login {
          ...
        }
        service managesieve {
          ...
        }

    Note that you should disallow access to port number 4190 using firewall rules. I recommend using ufw for simple iptables management.  

4.  Edit `/etc/dovecot/conf.d/15-lda.conf`

        protocol lda {
          mail_plugins = $mail_plugins sieve
        }

5.  Edit `/etc/roundcube/main.inc.php`

        $rcmail_config['plugins'] = array(
          'managesieve'
        );

6.  Edit `/etc/roundcube/plugins/managesieve/config.inc.php`

        <?php
          $rcmail_config['managesieve_host'] = 'localhost';
        ?>

7.  Restart services:  
`service dovecot restart`  
`service apache2 restart`  

8.  Login to roundcube and test filters. I created one that activates when email subject is equal to ‘test’ then moves it to another folder.

Happy administration!
