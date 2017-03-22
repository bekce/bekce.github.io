---
layout:     post
title:      Building My Mail Server
date:       2015-02-27
categories: sysadmin
---

I have built a multi-domain email server on Ubuntu with postfix, dovecot, mysql, roundcube and spamassassin 
by following [this guide](https://www.exratione.com/2014/05/a-mailserver-on-ubuntu-1404-postfix-dovecot-mysql/).
This post is a cut-through refined version from the exact steps that I followed, written while executing.
I hope it will be useful to some people: 

_2017 Update: I wouldn't go through these steps now and just use a hassle-free [prebuilt docker image](https://github.com/bekce/poste.io)._

Let's start.

	apt-get update
	apt-get upgrade --assume-yes

**1. Start with setting hostname**

	apt-get install --assume-yes systemd-services
	hostnamectl set-hostname mail.example.com
	nano /etc/hosts #put new hostname and static ip
	
We'll replace these with trusted ssl cert later on: 
	
	apt-get install --assume-yes ssl-cert
	make-ssl-cert generate-default-snakeoil --force-overwrite

**2. Install lamp server**

	apt-get install --assume-yes lamp-server^
	apt-get install --assume-yes php-apc php5-mcrypt php5-memcache php5-curl php5-gd php-xml-parser
	ln -sf ../../mods-available/mcrypt.ini /etc/php5/apache2/conf.d/20-mcrypt.ini
	
**3. Configure PHP (8)**

The default configuration settings for PHP and the additional packages mentioned above are sufficient for most casual usage. So unless you have something complicated or high-powered in mind, you should probably only change the expose_php setting in `/etc/php5/apache2/php.ini`. Set it to "Off":
	
		expose_php = Off

**4. Configure Apache (9)**

Firstly configure the following lines in `/etc/apache2/conf-enabled/security.conf` to minimize the information that Apache gives out in its response headers:
	
		ServerTokens Prod
		 
		ServerSignature Off
		
Edit these lines in `/etc/apache2/mods-available/ssl.conf` to ensure that protocols that are no longer secure are not used:
 		
		SSLCipherSuite HIGH:MEDIUM:!aNULL:!MD5
		SSLProtocol all -SSLv3

Put these in `/etc/apache2/sites-available/000-default.conf` 

		<Directory "/var/www/html">
			Options FollowSymLinks
			AllowOverride All
		</Directory>
		LogLevel warn		
	
Put these in `/etc/apache2/sites-available/default-ssl.conf`

        # seb:
        <Directory "/var/www/html">
          Options FollowSymLinks
          AllowOverride All
          SSLOptions +StdEnvVars
        </Directory>
        
        SSLCertificateFile /etc/ssl/private/sebworks.com.crt
        SSLCertificateKeyFile /etc/ssl/private/sebworks.com.key
		SSLCertificateChainFile /etc/ssl/private/sub.class1.server.ca.pem
 
Put your keys and chain file under `/etc/ssl/private`, then

	chmod -R 600 /etc/ssl/private
	
Edit `/var/www/html/.htaccess`
 
		RewriteEngine On
		# Redirect all HTTP traffic to HTTPS.
		RewriteCond %{HTTPS} !=on
		RewriteRule ^/?(.*) https://%{SERVER_NAME}/$1 [R,L]
		
	service apache2 restart
	
**5. Install more packages**

	apt-get install --assume-yes memcached
	
Choose Internet Site and default hostname (`sigma.sebworks.com`). 
Select no for dovecot ssl certificate question.

	apt-get install --assume-yes mail-server^

Install other stuff. Note that I omit postgrey because we don't need that much protection. 

	apt-get install --assume-yes \
		postfix-mysql \
		dovecot-mysql \
		amavis \
		clamav \
		clamav-daemon \
		spamassassin \
		php5-imap
  ln -sf ../../mods-available/imap.ini /etc/php5/apache2/conf.d/20-imap.ini
  service apache2 restart

I have removed some unnecessary packages from original doc, these are OK:

	apt-get install --assume-yes \
		pyzor \
		cabextract \
		p7zip-full \
		unzip \
		unrar-free \
		zip

**6. Setup mysql tables**

	mysql -uroot -p
	create database mail;
	grant all on mail.* to 'mail'@'localhost' identified by 'secret';

**7. Install postfix admin**

	cd /root
	wget http://downloads.sourceforge.net/project/postfixadmin/postfixadmin/postfixadmin-2.3.7/postfixadmin-2.3.7.tar.gz
	tar -xf postfixadmin-2.3.7.tar.gz
	rm -f postfixadmin-2.3.7.tar.gz
	mv postfixadmin-2.3.7 /var/www/html/postfixadmin
	chown -R www-data:www-data /var/www/html/postfixadmin

Follow [original guide](https://www.exratione.com/2014/05/a-mailserver-on-ubuntu-1404-postfix-dovecot-mysql/) 13 and 14 here. 
Put .htaccess at the end to block setup.php.
Create accounts.

My conflicting changes to `/var/www/html/postfixadmin/config.inc.php`: 

		$CONF['show_footer_text'] = 'NO';
		$CONF['domain_path'] = 'YES';
		$CONF['domain_in_mailbox'] = 'NO';

**8. Create vmail user**
	
	useradd -r -u 150 -g mail -d /var/vmail -s /sbin/nologin -c "Virtual maildir handler" vmail
	mkdir /var/vmail
	chmod 770 /var/vmail
	chown vmail:mail /var/vmail
	
**9. Dovecot configuration**

Follow [original guide](https://www.exratione.com/2014/05/a-mailserver-on-ubuntu-1404-postfix-dovecot-mysql/) 16 here. 

My conflicting changes to `/etc/dovecot/conf.d/10-auth.conf`: 
		
		auth_default_realm = sebworks.com

**10. Configure Amavis, ClamAV, and SpamAssassin**

Follow [original guide](https://www.exratione.com/2014/05/a-mailserver-on-ubuntu-1404-postfix-dovecot-mysql/) 17 here. 

**11. Configure Postfix**

Convenience bash commands for editing various files. Change MAILPASSWORD first. 

	export MAILPASSWORD=secret
	cat << EOF > /etc/postfix/mysql_virtual_alias_domainaliases_maps.cf
	user = mail
	password = $MAILPASSWORD
	hosts = 127.0.0.1
	dbname = mail
	query = SELECT goto FROM alias,alias_domain
	  WHERE alias_domain.alias_domain = '%d'
	  AND alias.address=concat('%u', '@', alias_domain.target_domain)
	  AND alias.active = 1
	EOF

	cat << EOF > /etc/postfix/mysql_virtual_alias_maps.cf
	user = mail
	password = $MAILPASSWORD
	hosts = 127.0.0.1
	dbname = mail
	table = alias
	select_field = goto
	where_field = address
	additional_conditions = and active = '1'
	EOF

	cat << EOF > /etc/postfix/mysql_virtual_domains_maps.cf
	user = mail
	password = $MAILPASSWORD
	hosts = 127.0.0.1
	dbname = mail
	table = domain
	select_field = domain
	where_field = domain
	additional_conditions = and backupmx = '0' and active = '1'
	EOF

	cat << EOF > /etc/postfix/mysql_virtual_mailbox_domainaliases_maps.cf
	user = mail
	password = $MAILPASSWORD
	hosts = 127.0.0.1
	dbname = mail
	query = SELECT maildir FROM mailbox, alias_domain
	  WHERE alias_domain.alias_domain = '%d'
	  AND mailbox.username=concat('%u', '@', alias_domain.target_domain )
	  AND mailbox.active = 1
	EOF

	cat << EOF > /etc/postfix/mysql_virtual_mailbox_maps.cf
	user = mail
	password = $MAILPASSWORD
	hosts = 127.0.0.1
	dbname = mail
	table = mailbox
	select_field = CONCAT(domain, '/', local_part)
	where_field = username
	additional_conditions = and active = '1'
	EOF

	cat << EOF > /etc/postfix/header_checks
	/^User-Agent:/               IGNORE
	/^X-Mailer:/                 IGNORE
	/^X-Originating-IP:/         IGNORE
	EOF

I have changed much of postfix settings from the original guide so don't use it. 

main.cf: 

	cp /etc/postfix/main.cf /etc/postfix/main.cf.original

	cat << 'EOF' > /etc/postfix/main.cf 
	# See /usr/share/postfix/main.cf.dist for a commented, more complete version

	# SASL parameters
	# ---------------------------------

	# Use Dovecot to authenticate.
	smtpd_sasl_type = dovecot
	# Referring to /var/spool/postfix/private/auth
	smtpd_sasl_path = private/auth
	smtpd_sasl_auth_enable = yes
	smtpd_sasl_security_options = noanonymous
	smtpd_sasl_local_domain =
	smtpd_sasl_authenticated_header = yes

	# TLS parameters
	# ---------------------------------

	smtpd_tls_auth_only = yes
	smtpd_tls_security_level = may
	smtp_tls_security_level = may
	smtpd_tls_cert_file=/etc/ssl/private/sebworks.com.crt
	smtpd_tls_key_file=/etc/ssl/private/sebworks.com.key
	smtpd_tls_CAfile=/etc/ssl/private/sub.class1.server.ca.pem
	smtpd_tls_mandatory_protocols=!SSLv2,!SSLv3
	smtp_tls_note_starttls_offer = yes
	smtpd_tls_loglevel = 1
	smtpd_tls_received_header = yes
	smtpd_tls_session_cache_timeout = 3600s
	tls_random_source = dev:/dev/urandom
	smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
	smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
	smtpd_sasl_tls_security_options = noanonymous

	# General parameters
	# ----------------------------------

	myhostname = sigma.sebworks.com
	mydestination =
	# By default, mynetworks_style controls default behavior of mynetworks parameter. They're mutually exclusive. 
	# If you have a separate web server that sends outgoing mail through this mailserver, you may want to add its IP address here
	# mynetworks = 108.61.190.170/32, 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
	# mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
	mynetworks_style = host

	smtpd_sender_restrictions = permit_mynetworks, permit_sasl_authenticated, warn_if_reject reject_non_fqdn_sender, reject_unknown_sender_domain
	smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
	smtpd_data_restrictions = reject_unauth_pipelining

	# Virtual mail
	# ---------------------------------------

	virtual_mailbox_base = /var/vmail
	virtual_mailbox_maps = mysql:/etc/postfix/mysql_virtual_mailbox_maps.cf, mysql:/etc/postfix/mysql_virtual_mailbox_domainaliases_maps.cf
	virtual_uid_maps = static:150
	virtual_gid_maps = static:8
	virtual_alias_maps = mysql:/etc/postfix/mysql_virtual_alias_maps.cf, mysql:/etc/postfix/mysql_virtual_alias_domainaliases_maps.cf
	virtual_mailbox_domains = mysql:/etc/postfix/mysql_virtual_domains_maps.cf

	# Misc parameters
	# ---------------------------------

	# The first text sent to a connecting process.
	smtpd_banner = $myhostname ESMTP $mail_name
	biff = no
	# appending .domain is the MUA's job.
	append_dot_mydomain = no
	# Tell postfix to hand off mail to the definition for dovecot in master.cf
	virtual_transport = dovecot
	dovecot_destination_recipient_limit = 1
	# Use amavis for virus and spam scanning
	content_filter = amavis:[127.0.0.1]:10024
	# Getting rid of unwanted headers. See: https://posluns.com/guides/header-removal/
	header_checks = regexp:/etc/postfix/header_checks
	enable_original_recipient = no
	mailbox_size_limit = 0
	recipient_delimiter = +

	delay_warning_time = 1h
	unknown_local_recipient_reject_code = 450
	maximal_queue_lifetime = 7d
	minimal_backoff_time = 1000s
	maximal_backoff_time = 8000s
	smtp_helo_timeout = 60s
	smtpd_soft_error_limit = 3
	smtpd_hard_error_limit = 12
	smtpd_delay_reject = yes
	disable_vrfy_command = yes
	smtpd_helo_required = yes

	EOF
	
master.cf: 

	cp /etc/postfix/master.cf /etc/postfix/master.cf.original
	cat << 'EOF' > /etc/postfix/master.cf
	
	smtp      inet  n       -       -       -       -       smtpd
	smtps     inet  n       -       -       -       -       smtpd
	  -o syslog_name=postfix/smtps
	  -o smtpd_tls_wrappermode=yes
	pickup    unix  n       -       -       60      1       pickup
	  -o content_filter=
	  -o receive_override_options=no_header_body_checks
	cleanup   unix  n       -       -       -       0       cleanup
	qmgr      unix  n       -       n       300     1       qmgr
	#qmgr     unix  n       -       n       300     1       oqmgr
	tlsmgr    unix  -       -       -       1000?   1       tlsmgr
	rewrite   unix  -       -       -       -       -       trivial-rewrite
	bounce    unix  -       -       -       -       0       bounce
	defer     unix  -       -       -       -       0       bounce
	trace     unix  -       -       -       -       0       bounce
	verify    unix  -       -       -       -       1       verify
	flush     unix  n       -       -       1000?   0       flush
	proxymap  unix  -       -       n       -       -       proxymap
	proxywrite unix -       -       n       -       1       proxymap
	smtp      unix  -       -       -       -       -       smtp
	relay     unix  -       -       -       -       -       smtp
	#       -o smtp_helo_timeout=5 -o smtp_connect_timeout=5
	showq     unix  n       -       -       -       -       showq
	error     unix  -       -       -       -       -       error
	retry     unix  -       -       -       -       -       error
	discard   unix  -       -       -       -       -       discard
	local     unix  -       n       n       -       -       local
	virtual   unix  -       n       n       -       -       virtual
	lmtp      unix  -       -       -       -       -       lmtp
	anvil     unix  -       -       -       -       1       anvil
	scache    unix  -       -       -       -       1       scache
	maildrop  unix  -       n       n       -       -       pipe
	  flags=DRhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
	uucp      unix  -       n       n       -       -       pipe
	  flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
	ifmail    unix  -       n       n       -       -       pipe
	  flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
	bsmtp     unix  -       n       n       -       -       pipe
	  flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
	scalemail-backend unix	-	n	n	-	2	pipe
	  flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
	mailman   unix  -       n       n       -       -       pipe
	  flags=FR user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py
	  ${nexthop} ${user}
	# The next two entries integrate with Amavis for anti-virus/spam checks.
	amavis      unix    -       -       -       -       3       smtp
	  -o smtp_data_done_timeout=1200
	  -o smtp_send_xforward_command=yes
	  -o disable_dns_lookups=yes
	  -o max_use=20
	127.0.0.1:10025 inet    n       -       -       -       -       smtpd
	  -o content_filter=
	  -o local_recipient_maps=
	  -o relay_recipient_maps=
	  -o smtpd_restriction_classes=
	  -o smtpd_delay_reject=no
	  -o smtpd_client_restrictions=permit_mynetworks,reject
	  -o smtpd_helo_restrictions=
	  -o smtpd_sender_restrictions=
	  -o smtpd_recipient_restrictions=permit_mynetworks,reject
	  -o smtpd_data_restrictions=reject_unauth_pipelining
	  -o smtpd_end_of_data_restrictions=
	  -o mynetworks=127.0.0.0/8
	  -o smtpd_error_sleep_time=0
	  -o smtpd_soft_error_limit=1001
	  -o smtpd_hard_error_limit=1000
	  -o smtpd_client_connection_count_limit=0
	  -o smtpd_client_connection_rate_limit=0
	  -o receive_override_options=no_header_body_checks,no_unknown_recipient_checks
	# Integration with Dovecot - hand mail over to it for local delivery, and
	# run the process under the vmail user and mail group.
	dovecot      unix   -        n      n       -       -   pipe
	  flags=DRhu user=vmail:mail argv=/usr/lib/dovecot/dovecot-lda -d $(recipient)
	
	EOF
	
**Test server** 

Restart everything and test mail server functionality 

	service postfix restart
	service spamassassin restart
	service clamav-daemon restart
	service amavis restart
	service dovecot restart
	
**Install roundcube**

Accept configuring mysql database for roundcube. 

	apt-get install --assume-yes roundcube
	apt-get install --assume-yes roundcube-plugins
	apt-get install --assume-yes roundcube-plugins-extra

Make these changes to `/etc/roundcube/main.inc.php` (change in their proper lines) 

		$rcmail_config['default_host'] = 'localhost';
		$rcmail_config['force_https'] = true;
		$rcmail_config['imap_cache'] = 'memcache';
		$rcmail_config['session_storage'] = 'memcache';
		$rcmail_config['memcache_hosts'] = array( 'localhost:11211' );

	ln -s /var/lib/roundcube /var/www/html/mail

**That's about all for now!**
