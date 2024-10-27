# mailserver

## component

Dovecot + Postfix + Roundcube

## Scheme





## describe scheme

tip: those server **not support exchange protocol** because exchange is develop from microsoft.

1. `Dovecot` is the core of the system; any operation requires the Dovecot service.
2. `Postfix` is used for sending and receiving mail.
3. `Roundcube` is only for the website."
4. `db` for save domain informations and user password you can change to other components
5. `storage` as db  you can change to other components

## Set Up

Use mysql 5.7 to start db. And set a db and table scheme

create db that dovecot will use those db

```sql
CREATE DATABASE mail CHARACTER SET utf8 COLLATE utf8_general_ci;

use mail;
CREATE TABLE users (
 id INT UNSIGNED AUTO_INCREMENT,
 userid VARCHAR(128) NOT NULL unique,
 domain VARCHAR(128) NOT NULL,
 password VARCHAR(500) NOT NULL,
 claimant VARCHAR(20) NOT NULL,
 description VARCHAR(200) NOT NULL,
 primary key(id)
);

# inster a item
INSERT INTO `users` VALUES (19, 'usermail', 'example.com', '{ARGON2ID}$argon2id$v=19$m=65536,t=3,p={ARGON2ID}$argon2id$v=19$m=65536,t=3,p=1$RmLSeYCr45SE84sqJqv/wA$jGB2u2M9TjZckWR+McZwJredi3Tzqlx3+cipgEcE4BI', 'admin', 'for test and password is 1234qwer@WSX');
```

create db that roundcube  will use those db

```sql
CREATE DATABASE roundcube CHARACTER SET utf8 COLLATE utf8_general_ci;
```

Last dns. require mx and A recode. I won't talk about those

## dovecot

I had configured some file dir like those

>runningconfig/
>├── cas
>│   ├── example.com.crt
>│   └──  example.com.key
>├── conf.d
>│   ├── auth.conf
>│   ├── mail.conf
>│   ├── protocols.conf
>│   └── ssl.conf
>├── dovecot.conf
>├── ext
>│   └── sql.conf.ext
>├── readme
>└── sqlfile

### dovecot.conf

This file is main file 

```
# I use docker run so I set those logs to run in the foreground. log require sent to  /dev/stdout is stout
log_path = /dev/stdout
debug_log_path = /dev/stdout
# dovecot  run as user, cat /etc/passwd can got those
mail_uid = 100
mail_gid = 103
# as mail_uid value
first_valid_uid = 100
# load other config
!include conf.d/*.conf

#  dovecot -c /config/dovecot.conf -F
```

### protocols.conf

```
protocols = imap pop3 lmtp
login_trusted_networks = 0.0.0.0/0

# imap
service imap-login {
  inet_listener imap {
    address = *
    port = 143
  }
  inet_listener imaps {
    address = *
    port = 993
    ssl = yes
  }
}

# pop3
service pop3-login {
  inet_listener pop3 {
    address = *
    port = 110
  }
  inet_listener pop3s {
    address = *
    port = 995
    ssl = yes
  }
}

# lmtp 
service lmtp {
   inet_listener lmtp {
      address = *
      port = 24
      # I am not test that
      # ssl = yes
   }
}
```

### mail.conf

```
# the mail will save the location %d domain %n username
mail_location = maildir:/mail/%d/%n

# who can read write /mail/%d/%n this dir
mail_privileged_group = vmail

namespace inbox {
  inbox = yes
}

protocol !indexer-worker {
  mail_vsize_bg_after_count = 0
}

namespace inbox {
  mailbox Archive {
    auto = subscribe
    special_use = \Archive
  }
  mailbox Drafts {
    auto = subscribe
    special_use = \Drafts
  }
  mailbox Junk {
    auto = subscribe
    special_use = \Junk
  }
  mailbox Trash {
    auto = subscribe
    special_use = \Trash
  }
  mailbox Sent {
    auto = subscribe
    special_use = \Sent
  }
}
```

### auth.conf

```
# require user must use plaintext auth
disable_plaintext_auth = no
# debug
auth_verbose = yes
auth_debug_passwords = yes
auth_verbose_passwords = plain

# how to auth
auth_mechanisms = plain login

# jwhere find user password
passdb {
  driver = sql
  args = /runningconfig/ext/sql.conf.ext
  # args = ext/sql.conf.ext
}

# where find user db
userdb {
 driver = sql
 # override_fields = home=/var/mail/%u mail=maildir:/var/mail/%u/Maildir
 args = /runningconfig/ext/sql.conf.ext
 # args = uid=vmail gid=vmail home=/usermail/%d/%n
}

# auth port listen
service auth {
  # https://wiki.dovecot.org/Services#inet_listeners
  inet_listener auth {
    address = *
    port = 4523
  }
}
```

### sql.conf.ext

```

driver = mysql
default_pass_scheme = ARGON2ID
connect = host=192.168.121.132 port=3306 dbname=mail user=root password=1234567890
password_query = SELECT userid AS username, domain, password FROM users WHERE userid = '%n' AND domain = '%d'
user_query = SELECT userid FROM users WHERE userid = '%n' AND domain = '%d'
iterate_query = SELECT userid AS username, domain FROM users
```

### ssl.conf

```nginx
ssl = required
ssl_prefer_server_ciphers = yes
ssl_cert = /runningconfig/cas/example.com.key
ssl_key = /runningconfig/cas/example.com.crt
```

### start

```
dovecot -c /config/dovecot.conf -F
```

## postfix

main.cf

```nginx
#  I use docker run so I set those logs to run in the foreground. log require sent to  /dev/stdout is stout
maillog_file=/dev/stdout
# mx code is smtp.example.com
myhostname=smtp.example.com
# ingress domain
mydomain=example.com
myorigin=$mydomain
#  when server received some mail then erver will catch  XXXX@example.com mail. other will have other logic to porcess
mydestination=$mydomain
# service in all interfaces
inet_interfaces=all

# Trust networks that send mail without authentication  mynetworks is better than mynetworks_style
# mynetworks_style=host
mynetworks=192.168.1.0/24, 10.0.0.0/8, 127.0.0.0/8, 0.0.0.0/0

# allow relay domains.   when server received some mail is not  XXXX@example.com. then allow send to other domail mail server. Those is not safe configure
# relay_domains=$mydomain

# -------------auth----------------
# for auth use is match
smtpd_sender_login_maps=mysql:/config/sender_map.cf
#  auth use sasl
smtpd_sasl_auth_enable=yes
# use dovecot auth
smtpd_sasl_type=dovecot
# dovecot auth ip and port
smtpd_sasl_path=inet:192.168.121.132:4523
# how to auth  from dovecot
# noanonymous: must give username and passoword to auth
# noplaintext: 
smtpd_sasl_security_options=noanonymous
smtpd_reject_unlisted_sender=yes

# allow sender
smtpd_sender_restrictions=
	# as configure
	reject_unknown_sender_domain,
	# reject when login username is aaa but send as  bbb require smtpd_sender_login_maps 
	reject_authenticated_sender_login_mismatch,
	# reject when domain is @example.com but send as  not @example.com
	reject_non_fqdn_sender,
	# allow sasl login user
	permit_sasl_authenticated,
	# reject other
	reject

# all recipien mail
smtpd_recipient_restrictions=
  # all mail from those network
  permit_mynetworks,
  # reject no domail
  reject_non_fqdn_recipient,
  # reject invalid helo hostname may be is a attack
  reject_invalid_helo_hostname,
  permit

# smtps
smtpd_tls_auth_only=no
broken_sasl_auth_clients=yes
smtpd_use_tls=yes
smtpd_tls_CAfile=/ca/ca.crt
smtpd_tls_cert_file=/ca/server.crt
smtpd_tls_key_file=/ca/server.private.key.pem
smtpd_tls_session_cache_database=lmdb:/var/run/smtpd_tls_session_cache
# other configure
.....
virtual_mailbox_domains = example.com
# dovecot port
virtual_transport = lmtp:192.168.121.132:24
```

### master.cf

```
smtps     inet  n       -       n       -       -       smtpd -o smtpd_tls_wrappermode=yes -o smtpd_sasl_auth_enable=yes
smtp      inet  n       -       n       -       -       smtpd
pickup    unix  n       -       n       60      1       pickup
cleanup   unix  n       -       n       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
tlsmgr    unix  -       -       n       1000?   1       tlsmgr
rewrite   unix  -       -       n       -       -       trivial-rewrite
bounce    unix  -       -       n       -       0       bounce
defer     unix  -       -       n       -       0       bounce
trace     unix  -       -       n       -       0       bounce
verify    unix  -       -       n       -       1       verify
flush     unix  n       -       n       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       n       -       -       smtp
relay     unix  -       -       n       -       -       smtp
  -o syslog_name=postfix/$service_name
showq     unix  n       -       n       -       -       showq
error     unix  -       -       n       -       -       error
retry     unix  -       -       n       -       -       error
discard   unix  -       -       n       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       n       -       -       lmtp
anvil     unix  -       -       n       -       1       anvil
scache    unix  -       -       n       -       1       scache
postlog   unix-dgram n  -       n       -       1       postlogd
```

### sender_map.cf

```
# %u is mail prefix, %d is mail domain 
hosts=host:port
user=root
password=1234567890
dbname=mail
query=SELECT CONCAT (userid, '@' , domain ) FROM users WHERE userid='%u' AND domain='%d'
```

## roundcube

docker web set: https://hub.docker.com/r/roundcube/roundcubemail/

```bash
IMAP_SERVER="192.168.121.132"
IMAP_PORT="143"

SMTP_SERVER="192.168.121.132"
SMTP_PORT="25"

DB_TYPE='mysql'
DB_HOST='192.168.121.13'
DB_PORT="3306"
DB_USER="root"
DB_PASSWORD="1234567890"
DB_NAME="roundcubemail"

nerdctl run -it --rm \
-p 0.0.0.0:8000:80 \
-e ROUNDCUBEMAIL_DEFAULT_HOST=$IMAP_SERVER \
-e ROUNDCUBEMAIL_DEFAULT_PORT=$IMAP_PORT \
-e ROUNDCUBEMAIL_SMTP_SERVER=$SMTP_SERVER \
-e ROUNDCUBEMAIL_SMTP_PORT=$SMTP_PORT \
-e ROUNDCUBEMAIL_DB_TYPE=$DB_TYPE \
-e ROUNDCUBEMAIL_DB_HOST=$DB_HOST \
-e ROUNDCUBEMAIL_DB_PORT=$DB_PORT \
-e ROUNDCUBEMAIL_DB_USER=$DB_USER \
-e ROUNDCUBEMAIL_DB_PASSWORD=$DB_PASSWORD \
-e ROUNDCUBEMAIL_DB_NAME=roundcubemail \
roundcubeorg/roundcubemail:latest-apache
```

