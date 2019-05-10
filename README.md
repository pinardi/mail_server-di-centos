# mail_server-di-centos


## Setting Hostname
Ganti hostname server anda menjadi mail.[NAMADOMAIN].com

'''bash
hostnamectl --static set-hostname mailxen.minion.com
'''

Hapus Mail Server Lain
Hapus exim dan sendmail

yum remove exim* sendmail*
Install MariaDB
Install MariaDB dengan perintah
Setting Hostname
Ganti hostname server anda menjadi mail.[NAMADOMAIN].com

hostnamectl --static set-hostname mail.jaranguda.com
Hapus Mail Server Lain
Hapus exim dan sendmail

yum remove exim* sendmail*
Install MariaDB
Install MariaDB dengan perintah

buat database dan user untuk mail.

create database email;
GRANT ALL PRIVILEGES ON email.* TO "dbmail"@"localhost" IDENTIFIED BY "sFRjKXVkUef3VHxTXiLT";
Buat tabel di database
CREATE TABLE domains (domain varchar(50) NOT NULL, PRIMARY KEY (domain) );
CREATE TABLE forwardings (source varchar(80) NOT NULL, destination TEXT NOT NULL, PRIMARY KEY (source) );
CREATE TABLE users (email varchar(80) NOT NULL, password varchar(20) NOT NULL, PRIMARY KEY (email) );
CREATE TABLE transport ( domain varchar(128) NOT NULL default '', transport varchar(128) NOT NULL default '', UNIQUE KEY domain (domain) );
Generate SSL Self Signed
Untuk sementara kita akan menggunakan self signed SSL, nantinya bisa diganti dengan SSL berbayar atau Lets Encrypt

mkdir /etc/postfix/ssl;
cd /etc/postfix/ssl;
openssl req -x509 -nodes -newkey rsa:2048 -keyout mail.xxx.com.key -out mail.xxx.com.crt -nodes -days 3650
output perintah diatas

$ openssl req -x509 -nodes -newkey rsa:2048 -keyout mail.xxx.com.key -out mail.indounix.com.crt -nodes -days 3650
Generating a 2048 bit RSA private key
.............................................................................................+++
..................................................................+++
writing new private key to 'mail.xxx.com.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:ID
State or Province Name (full name) []:Jakarta
Locality Name (eg, city) [Default City]:Jakarta
Organization Name (eg, company) [Default Company Ltd]:SSL
Organizational Unit Name (eg, section) []:SSL
Common Name (eg, your name or your server's hostname) []:mail.xxx.com
Email Address []:t@linux.com
[root@mail.xxx.com] /etc/postfix/ssl 
$ ls
mail.xxx.com.crt  mail.xxx.com.key
Install dan Konfigurasi Postfix
Install postfix

yum -y --enablerepo=centosplus install postfix
Database Mail
Buat 4 file dibawah ini
/etc/postfix/mysql-virtual_domains.cf

user = dbmail
password = sFRjKXVkUef3VHxTXiLT
dbname = email
query = SELECT domain AS virtual FROM domains WHERE domain='%s'
hosts = 127.0.0.1
/etc/postfix/mysql-virtual_email2email.cf

user = dbmail
password = sFRjKXVkUef3VHxTXiLT
dbname = email
query = SELECT email FROM users WHERE email='%s'
hosts = 127.0.0.1
/etc/postfix/mysql-virtual_forwardings.cf

user = dbmail
password = sFRjKXVkUef3VHxTXiLT
dbname = email
query = SELECT destination FROM forwardings WHERE source='%s'
hosts = 127.0.0.1
/etc/postfix/mysql-virtual_mailboxes.cf

user = dbmail
password = sFRjKXVkUef3VHxTXiLT
dbname = email
query = SELECT CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/') FROM users WHERE email='%s'
hosts = 127.0.0.1
Konfigurasi Postfix
buka file /etc/postfix/master.cf lihat bagian

submission inet n	-	n	-	-	smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
hilangkan tanda pagar # didepannya.

Edit file /etc/postfix/main.cf

#myhostname = host.domain.tld
#mydomain = domain.tld
#myorigin = $myhostname
#inet_interfaces = all
ubah menjadi

myhostname = mail.indounix.com
mydomain = indounix.com
myorigin = $myhostname
inet_interfaces = all
tambahkan dibagian paling bawah

smtpd_use_tls = yes
smtpd_tls_auth_only = yes
smtpd_tls_cert_file = /etc/postfix/ssl/mail.indounix.com.crt
smtpd_tls_key_file = /etc/postfix/ssl/mail.indounix.com.key
smtpd_tls_mandatory_protocols=!SSLv2,!SSLv3
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
 
virtual_transport = lmtp:unix:private/dovecot-lmtp
virtual_alias_maps = proxy:mysql:/etc/postfix/mysql-virtual_forwardings.cf, mysql:/etc/postfix/mysql-virtual_email2email.cf
virtual_mailbox_domains = proxy:mysql:/etc/postfix/mysql-virtual_domains.cf
virtual_mailbox_maps = proxy:mysql:/etc/postfix/mysql-virtual_mailboxes.cf
virtual_mailbox_base = /home/vmail
 
#SASL
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes
smtpd_sasl_authenticated_header = yes
 
#Limit
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
Restart Postfix

systemctl restart  postfix.service
Install dan Konfigurasi Dovecot
yum install dovecot dovecot-mysql
Buat folder penyimpanan email di /home/vmail

groupadd -g 5000 vmail
useradd -g vmail -u 5000 vmail -d /home/vmail -m
chown -R vmail.vmail /home/vmail
Edit file /etc/dovecot/conf.d/10-mail.conf, ubah

#separator = 
#mail_location =
menjadi

separator = .
mail_location = maildir:/home/vmail/%d/%n/Maildir
Edit file /etc/dovecot/conf.d/10-auth.conf, ubah

auth_mechanisms = plain
!include auth-system.conf.ext
#!include auth-sql.conf.ext
menjadi

auth_mechanisms = plain login
#!include auth-system.conf.ext
!include auth-sql.conf.ext
Edit file /etc/dovecot/conf.d/10-master.conf, ubah

#unix_listener /var/spool/postfix/private/auth {
#  mode = 0666
#}
menjadi

unix_listener /var/spool/postfix/private/auth {
  mode = 0660
  user = postfix
  group = postfix
}
Edit file /etc/dovecot/conf.d/10-ssl.conf, ubah

ssl = required
ssl_cert = </etc/pki/dovecot/certs/dovecot.pem
ssl_key = </etc/pki/dovecot/private/dovecot.pem
menjadi

ssl = yes
ssl_cert = </etc/postfix/ssl/mail.indounix.com.crt
ssl_key = </etc/postfix/ssl/mail.indounix.com.key
Edit file /etc/dovecot/conf.d/15-mailboxes.conf, ubah

  mailbox Drafts {
    special_use = \Drafts
  }
  mailbox Junk {
    special_use = \Junk
  }
  mailbox Trash {
    special_use = \Trash
  }
menjadi

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
Buat file /etc/dovecot/dovecot-sql.conf.ext

// 5000 disini sesuaikan waktu membuat user vmail
user_query = SELECT ('5000') as 'uid',('5000') as 'gid'
driver = mysql
connect = host=127.0.0.1 dbname=email user=email password=sFRjKXVkUef3VHxTXiLT
default_pass_scheme = SHA256-CRYPT
password_query = SELECT email as user, password FROM users WHERE email='%u';
Buka file /etc/dovecot/conf.d/10-master.conf, ubah

service lmtp {
  unix_listener lmtp {
    #mode = 0666
  }
 
  # Create inet listener only if you can't use the above UNIX socket
  #inet_listener lmtp {
    # Avoid making LMTP visible for the entire internet
    #address =
    #port =
  #}
}
menjadi

service lmtp {
 
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    group = postfix
    user = postfix
  }
 
}
jalankan dovecot

service dovecot restart
Agar perubahan di dovecot bisa dikenali oleh postfix, jalankan

postconf virtual_transport=lmtp:unix:private/dovecot-lmtp
lalu restart postfix

service postfix restart
Cek Port
Cek port yang terbuka di server netstat -tunlp

$ netstat -tunlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      15928/mysqld        
tcp        0      0 127.0.0.1:587           0.0.0.0:*               LISTEN      16376/master        
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      16376/master        
tcp        0      0 0.0.0.0:38400           0.0.0.0:*               LISTEN      4649/sshd           
tcp6       0      0 ::1:587                 :::*                    LISTEN      16376/master        
tcp6       0      0 ::1:25                  :::*                    LISTEN      16376/master        
tcp6       0      0 :::38400                :::*                    LISTEN      4649/sshd           
udp        0      0 127.0.0.1:323           0.0.0.0:*                           3201/chronyd        
udp6       0      0 ::1:323                 :::*                                3201/chronyd
port 587 dan 25 adalah port SMTP

Bisa juga dicek menggunakan telnet

yum install telnet -y
Coba konek ke port 25 dan 587

$ telnet localhost 25
Trying ::1...
Connected to localhost.
Escape character is '^]'.
220 mail.xxx.com ESMTP Postfix
quit
Connection closed by foreign host.
 
$ telnet localhost 587
Trying ::1...
Connected to localhost.
Escape character is '^]'.
220 mail.xxx.com ESMTP Postfix
quit
221 2.0.0 Bye
Connection closed by foreign host.
Tes Kirim Email
yum install mailx -y
contoh kita akan mengirim email ke gmail

$ mailx emailanda@gmail.com
Subject: Mail Server CentOS 7
ini isi email dari : Mail Server CentOS 7
Cek log dengan journal -f

Mar 3 13:16:18 mail postfix/smtpd[17210]: connect from localhost[127.0.0.1]
Mar 3 13:16:18 mail sendmail[17209]: STARTTLS=client, relay=[127.0.0.1], version=TLSv1/SSLv3, verify=FAIL, cipher=ECDHE-RSA-AES256-GCM-SHA384, bits=256/256
Mar 3 13:16:18 mail postfix/smtpd[17210]: C62506DA: client=localhost[127.0.0.1]
Mar 3 13:16:18 mail postfix/cleanup[17213]: C62506DA: message-id=<201703030616.v236GIaY017209@mail.indounix.com>
Mar 3 13:16:18 mail postfix/qmgr[17155]: C62506DA: from=, size=713, nrcpt=1 (queue active)
Mar 3 13:16:18 mail sendmail[17209]: v236GIaY017209: to=karonise@gmail.com, ctladdr=root (0/0), delay=00:00:00, xdelay=00:00:00, mailer=relay, pri=30270, relay=[127.0.0.1] [127.0.0.1], dsn=2.0.0, stat=Sent (Ok: queued as C62506DA)
Mar 3 13:16:18 mail postfix/smtpd[17210]: disconnect from localhost[127.0.0.1]
Mar 3 13:16:20 mail postfix/smtp[17214]: C62506DA: to=, relay=gmail-smtp-in.l.google.com[74.125.68.27]:25, delay=2, delays=0.06/0.02/0.47/1.4, dsn=2.0.0, status=sent (250 2.0.0 OK 1488521780 a3si9599213pgd.21 – gsmtp)
Mar 3 13:16:20 mail postfix/qmgr[17155]: C62506DA: removed
di Gmail
email dari postfix ke gmail

Tambah Domain
Untuk menambah domain baru login ke MySQL lalu eksekusi perintah

INSERT INTO domains (domain) VALUES ('jaranguda.com');
anda bisa menambah domain sebauak-banyaknya/unlimited. Ganti jaranguda.com dengan nama domain/subdomain anda

Buat Akun Email
Untuk menambah akun email baru login ke MySQL lalu eksekusi perintah

INSERT INTO users (email, password) VALUES ('nama.user@jaranguda.com', ENCRYPT('PASSWORD'));
ganti nama.user@jaranguda.com dengan email pilihan anda, dan PASSWORD dengan password anda. Anda bebas membuat password sepanjang mungkin > 50 karakter.
Bila anda ingin menambahkan email untuk domain yang belum ada di sistem, anda harus menambahkan domain terlebih dahulu.

Pembuatan akun email disini juga unlimited, jadi tidak perlu khawatir kehabisan akun email ;0

