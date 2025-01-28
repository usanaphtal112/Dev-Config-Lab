# Comprehensive Email Server Setup Guide for naphtal.tech
📧 A complete guide to setting up a production-ready email server

## Table of Contents
1. [Prerequisites & Planning](#prerequisites)
2. [Security Group Configuration](#security)
3. [DNS Setup](#dns)
4. [Core Software Installation](#installation)
5. [SSL Certificate Setup](#ssl)
6. [Postfix Configuration](#postfix)
7. [Virtual Users Setup](#virtual-users)
8. [Dovecot Configuration](#dovecot)
9. [Security Hardening](#security-hardening)
10. [Testing & Verification](#testing)
11. [Email Client Setup](#client-setup)
12. [Monitoring & Maintenance](#monitoring)
13. [Django/Flask Email Integration](#django-flask)
14. [Troubleshooting Guide](#troubleshooting)

<a name="prerequisites"></a>
## 1. Prerequisites & Planning 🔍

### Server Details
- EC2 Instance IP: 34.203.228.116
- Domain Name: naphtal.tech
- Mail Server Hostname: mail.naphtal.tech

### Required Resources
- Ubuntu Server 22.04 LTS
- Minimum 1GB RAM
- 20GB Storage
- Stable Internet Connection

### Access Requirements
- SSH access to EC2 instance
- Domain registrar access
- AWS Console access

<a name="security"></a>
## 2. Security Group Configuration 🛡️

### Required Ports
Create a new security group in AWS Console:

```plaintext
┌──────────────┬─────────┬────────────┬────────────────────────┐
│ Protocol     │ Port    │ Source     │ Purpose                │
├──────────────┼─────────┼────────────┼────────────────────────┤
│ TCP          │ 22      │ Your IP    │ SSH Access             │
│ TCP          │ 25      │ 0.0.0.0/0  │ SMTP                   │
│ TCP          │ 465     │ 0.0.0.0/0  │ SMTPS (Legacy Secure)  │
│ TCP          │ 587     │ 0.0.0.0/0  │ Submission (Modern)    │
│ TCP          │ 143     │ 0.0.0.0/0  │ IMAP                   │
│ TCP          │ 993     │ 0.0.0.0/0  │ IMAPS (Secure)         │
│ TCP          │ 80      │ 0.0.0.0/0  │ HTTP (Let's Encrypt)   │
│ TCP          │ 443     │ 0.0.0.0/0  │ HTTPS                  │
└──────────────┴─────────┴────────────┴────────────────────────┘
```

### Initial Server Access
```bash
# Set correct permissions for SSH key
chmod 400 your-key.pem

# Connect to EC2 instance
ssh -i your-key.pem ubuntu@34.203.228.116

# Verify connection
whoami  # Should return: ubuntu
```

<a name="dns"></a>
## 3. DNS Setup 🌐

### Required DNS Records
Log into your domain registrar's control panel and add these records:

```plaintext
┌─────────┬────────────────────────┬──────────────────────────────────┬───────┐
│ Type    │ Name                   │ Value                            │ TTL   │
├─────────┼────────────────────────┼──────────────────────────────────┼───────┤
│ A       │ mail.naphtal.tech.     │ 34.203.228.116                   │ 3600  │
│ A       │ naphtal.tech.          │ 34.203.228.116                   │ 3600  │
│ MX      │ naphtal.tech.          │ mail.naphtal.tech. (prio:10)     │ 3600  │
│ TXT     │ naphtal.tech.          │ v=spf1 ip4:34.203.228.116        │ 3600  │
│         │                        │ a mx ~all                        │       │
│ TXT     │ _dmarc.naphtal.tech.   │ v=DMARC1; p=none;                │ 3600  │
│         │                        │ rua=mailto:admin@naphtal.tech    │       │
└─────────┴────────────────────────┴──────────────────────────────────┴───────┘
```

### Verify DNS Configuration
```bash
# Install dig if not present
sudo apt install dnsutils -y

# Test each record
for record in A MX TXT; do
    echo "Testing $record record..."
    dig $record naphtal.tech
    echo "----------------------------------------"
done

# Test mail subdomain
dig A mail.naphtal.tech

# Test DMARC record
dig TXT _dmarc.naphtal.tech
```

Expected Output Example:
```plaintext
;; ANSWER SECTION:
naphtal.tech.        3600    IN      A       34.203.228.116
```

⚠️ Important: Wait at least 15-30 minutes for DNS propagation before proceeding.

<a name="installation"></a>
## 4. Core Software Installation 📦

### System Updates
```bash
# Update package lists and upgrade system
sudo apt update
sudo apt upgrade -y

# Install essential packages
sudo apt install \
    postfix postfix-mysql \
    dovecot-core dovecot-imapd \
    dovecot-lmtpd dovecot-mysql \
    mysql-server mailutils \
    opendkim opendkim-tools \
    fail2ban certbot \
    net-tools curl wget -y
```

### Postfix Initial Setup
During installation:
1. Choose "Internet Site" when prompted
2. Enter "naphtal.tech" as system mail name

### Verify Installation
```bash
# Check installed packages
dpkg -l | grep -E 'postfix|dovecot|mysql'

# Check service status
for service in postfix dovecot mysql; do
    echo "Checking $service status..."
    sudo systemctl status $service | grep Active
done
```

<a name="ssl"></a>
## 5. SSL Certificate Setup 🔒

### 5.1 Install Certbot
```bash
# Install Certbot and its dependencies
sudo apt update
sudo apt install certbot python3-certbot-nginx -y

# Stop any services using port 80
sudo systemctl stop nginx apache2 2>/dev/null
```

### 5.2 Obtain SSL Certificate
```bash
# Request certificate for mail domain
sudo certbot certonly --standalone \
    --preferred-challenges http \
    -d mail.naphtal.tech \
    --agree-tos \
    --email admin@naphtal.tech \
    --rsa-key-size 4096

# Verify certificate files
sudo ls -l /etc/letsencrypt/live/mail.naphtal.tech/
```

Expected Output:
```plaintext
total 4
lrwxrwxrwx 1 root root 42 Mar 20 12:00 cert.pem -> ../../archive/mail.naphtal.tech/cert1.pem
lrwxrwxrwx 1 root root 43 Mar 20 12:00 chain.pem -> ../../archive/mail.naphtal.tech/chain1.pem
lrwxrwxrwx 1 root root 47 Mar 20 12:00 fullchain.pem -> ../../archive/mail.naphtal.tech/fullchain1.pem
lrwxrwxrwx 1 root root 45 Mar 20 12:00 privkey.pem -> ../../archive/mail.naphtal.tech/privkey1.pem
```

### 5.3 Set Up Auto-Renewal
```bash
# Create renewal hook directory
sudo mkdir -p /etc/letsencrypt/renewal-hooks/post

# Create renewal script
sudo nano /etc/letsencrypt/renewal-hooks/post/reload-mail-services.sh
```

Add to the script:
```bash
#!/bin/bash

# Reload mail services after certificate renewal
systemctl reload postfix
systemctl reload dovecot

# Log the renewal
echo "Certificate renewed on $(date)" >> /var/log/letsencrypt-renewal.log
```

Set permissions:
```bash
# Make script executable
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/reload-mail-services.sh

# Test auto-renewal
sudo certbot renew --dry-run
```

### 5.4 Configure Certificate Permissions
```bash
# Allow mail services to read certificates
sudo chmod -R 755 /etc/letsencrypt/live/
sudo chmod -R 755 /etc/letsencrypt/archive/

# Verify permissions
ls -la /etc/letsencrypt/live/mail.naphtal.tech/
```

### 5.5 Update Service Configurations

Update Postfix SSL settings:
```bash
sudo postconf -e 'smtpd_tls_cert_file=/etc/letsencrypt/live/mail.naphtal.tech/fullchain.pem'
sudo postconf -e 'smtpd_tls_key_file=/etc/letsencrypt/live/mail.naphtal.tech/privkey.pem'
```

Update Dovecot SSL settings:
```bash
sudo sed -i 's|^ssl_cert.*|ssl_cert = </etc/letsencrypt/live/mail.naphtal.tech/fullchain.pem|' /etc/dovecot/conf.d/10-ssl.conf
sudo sed -i 's|^ssl_key.*|ssl_key = </etc/letsencrypt/live/mail.naphtal.tech/privkey.pem|' /etc/dovecot/conf.d/10-ssl.conf
```

### 5.6 Verify SSL Setup
```bash
# Test SSL certificate
openssl s_client -connect mail.naphtal.tech:993 -showcerts

# Test SMTP over SSL
openssl s_client -connect mail.naphtal.tech:465 -showcerts

# Check certificate expiration
echo | openssl s_client -servername mail.naphtal.tech \
    -connect mail.naphtal.tech:993 2>/dev/null | \
    openssl x509 -noout -dates
```

Common Issues and Solutions:

1. Certificate Renewal Failures:
```bash
# Check renewal logs
sudo tail -f /var/log/letsencrypt/letsencrypt.log

# Force renewal test
sudo certbot renew --force-renewal --dry-run
```

2. Permission Issues:
```bash
# Fix common permission problems
sudo chmod -R 755 /etc/letsencrypt/live/
sudo chmod -R 755 /etc/letsencrypt/archive/
sudo chown -R root:root /etc/letsencrypt/live/
sudo chown -R root:root /etc/letsencrypt/archive/
```

3. Service Reload Issues:
```bash
# Check service status after reload
sudo systemctl status postfix
sudo systemctl status dovecot

# Check logs for errors
sudo journalctl -u postfix --since "1 hour ago"
sudo journalctl -u dovecot --since "1 hour ago"
```

<a name="postfix"></a>
## 6. Postfix Configuration 📨

### 6.1 Backup Original Configuration
```bash
# Create backup directory
sudo mkdir -p /etc/mail/backup

# Backup original configurations with timestamp
sudo cp /etc/postfix/main.cf /etc/mail/backup/main.cf.$(date +%Y%m%d)
sudo cp /etc/postfix/master.cf /etc/mail/backup/master.cf.$(date +%Y%m%d)

# Verify backups
ls -lh /etc/mail/backup/
```

### 6.2 Main Configuration File
```bash
# Edit main.cf
sudo nano /etc/postfix/main.cf
```

📝 Configuration Breakdown:
```plaintext
# Basic Server Identity
# -------------------
myhostname = mail.naphtal.tech
myorigin = naphtal.tech
mydomain = naphtal.tech

# Network Settings
# --------------
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

# TLS/SSL Security
# --------------
# Certificate paths will be updated after Let's Encrypt setup
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.naphtal.tech/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/mail.naphtal.tech/privkey.pem
smtpd_use_tls = yes
smtpd_tls_auth_only = yes
smtpd_tls_security_level = may
smtp_tls_security_level = may

# Modern TLS Security Settings
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtp_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1

# Authentication
# ------------
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
broken_sasl_auth_clients = yes

# Virtual Domain and User Settings
# -----------------------------
virtual_mailbox_domains = hash:/etc/postfix/virtual-mailbox-domains
virtual_mailbox_maps = hash:/etc/postfix/virtual-mailbox-users
virtual_mailbox_base = /var/mail/vhosts
virtual_minimum_uid = 100
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000
virtual_transport = lmtp:unix:private/dovecot-lmtp

# SMTP Security Restrictions
# -----------------------
smtpd_recipient_restrictions =
    permit_sasl_authenticated,
    permit_mynetworks,
    reject_unauth_destination,
    reject_invalid_hostname,
    reject_non_fqdn_hostname,
    reject_non_fqdn_sender,
    reject_non_fqdn_recipient,
    reject_unknown_sender_domain,
    reject_unknown_recipient_domain,
    reject_rbl_client zen.spamhaus.org
```

### 6.3 Master Configuration File
```bash
# Edit master.cf
sudo nano /etc/postfix/master.cf
```

Add these service definitions after beginning of the file:
```plaintext
# Submission port 587
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (no)    (never) (100)
# ==========================================================================

# Standard SMTP (port 25)
smtp      inet  n       -       y       -       -       smtpd

# Submission port 587
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING

# SMTPS port 465
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

### 6.4 Verify Configuration
```bash
# Test Postfix configuration
sudo postfix check

# Test for syntax errors
sudo postconf -n

# Look for any error messages
sudo journalctl -u postfix
```

Expected Output:
```plaintext
postfix/postfix-script[xxxx]: warning: group or other writable: /etc/postfix/main.cf
postfix/postfix-script[xxxx]: starting the Postfix mail system
```

⚠️ Common Issues and Solutions:
1. Permission errors:
```bash
# Fix permissions
sudo chmod 644 /etc/postfix/main.cf
sudo chmod 644 /etc/postfix/master.cf
```

2. Certificate errors (before Let's Encrypt setup):
```bash
# Temporarily disable TLS until certificates are installed
sudo postconf -e 'smtpd_tls_security_level = none'
```

<a name="virtual-users"></a>
## 7. Virtual Users Setup 👥

### 7.1 Mail Directory Structure
```bash
# Create main mail directory
sudo mkdir -p /var/mail/vhosts/naphtal.tech

# Create vmail group and user
sudo groupadd -g 5000 vmail
sudo useradd -g vmail -u 5000 vmail -d /var/mail/vhosts -m

# Set ownership and permissions
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 770 /var/mail/vhosts

# Verify setup
ls -la /var/mail/vhosts
```

Expected Output:
```plaintext
drwxrwx--- 3 vmail vmail 4096 Mar 20 10:00 .
drwxr-xr-x 3 root  root  4096 Mar 20 10:00 ..
drwxrwx--- 2 vmail vmail 4096 Mar 20 10:00 naphtal.tech
```

### 7.2 Virtual Domain Configuration
```bash
# Create virtual domains file
sudo nano /etc/postfix/virtual-mailbox-domains
```

Add domain:
```plaintext
naphtal.tech OK
```

### 7.3 Virtual Users Setup
```bash
# Create virtual users file
sudo nano /etc/postfix/virtual-mailbox-users
```

Add users (adjust emails and paths as needed):
```plaintext
# Format: email_address    mailbox_path
admin@naphtal.tech        naphtal.tech/admin/
user1@naphtal.tech        naphtal.tech/user1/
user2@naphtal.tech        naphtal.tech/user2/
user3@naphtal.tech        naphtal.tech/user3/
user4@naphtal.tech        naphtal.tech/user4/
```

### 7.4 Create Mailbox Directories
```bash
# Create individual mailbox directories
for user in admin user1 user2 user3 user4; do
    sudo mkdir -p "/var/mail/vhosts/naphtal.tech/$user"
    sudo chown -R vmail:vmail "/var/mail/vhosts/naphtal.tech/$user"
    sudo chmod -R 700 "/var/mail/vhosts/naphtal.tech/$user"
done

# Verify directory structure
tree /var/mail/vhosts/naphtal.tech
```

Expected Output:
```plaintext
/var/mail/vhosts/naphtal.tech/
├── admin
├── user1
├── user2
├── user3
└── user4
```

### 7.5 Generate Postfix Lookup Tables
```bash
# Create database files
sudo postmap /etc/postfix/virtual-mailbox-domains
sudo postmap /etc/postfix/virtual-mailbox-users

# Verify database files
ls -l /etc/postfix/*.db
```

### 7.6 User Password Management
```bash
# Create password file
sudo nano /etc/dovecot/users
```

Add user credentials (use strong passwords!):
```plaintext
# Format: email:{PLAIN}password
admin@naphtal.tech:{PLAIN}StrongPass123!
user1@naphtal.tech:{PLAIN}UserPass456!
user2@naphtal.tech:{PLAIN}SecurePass789!
user3@naphtal.tech:{PLAIN}SafePass321!
user4@naphtal.tech:{PLAIN}GoodPass654!
```

Set secure permissions:
```bash
sudo chown -R vmail:dovecot /etc/dovecot/users
sudo chmod 640 /etc/dovecot/users
```

⚠️ Security Tips:
1. Use strong passwords (minimum 12 characters)
2. Include mix of uppercase, lowercase, numbers, and symbols
3. Store passwords securely
4. Consider using encrypted password hashes instead of PLAIN

### 7.7 Verify Setup
```bash
# Check permissions
find /var/mail/vhosts -type d -ls

# Test Postfix maps
postmap -q naphtal.tech /etc/postfix/virtual-mailbox-domains
postmap -q admin@naphtal.tech /etc/postfix/virtual-mailbox-users

# Check for errors
sudo tail -f /var/log/mail.log
```

Common Issues and Solutions:

1. Permission Denied Errors:
```bash
# Fix permissions if needed
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 700 /var/mail/vhosts
```

2. Database File Errors:
```bash
# Rebuild database files
sudo rm /etc/postfix/*.db
sudo postmap /etc/postfix/virtual-mailbox-domains
sudo postmap /etc/postfix/virtual-mailbox-users
```

3. User Directory Issues:
```bash
# Fix directory ownership
for user in admin user1 user2 user3 user4; do
    sudo mkdir -p "/var/mail/vhosts/naphtal.tech/$user"
    sudo chown -R vmail:vmail "/var/mail/vhosts/naphtal.tech/$user"
done
```

<a name="dovecot"></a>
## 8. Dovecot Configuration 📬

### 8.1 Backup Original Configuration
```bash
# Create backup of original configuration
sudo cp -r /etc/dovecot /etc/dovecot.backup.$(date +%Y%m%d)

# Verify backup
ls -la /etc/dovecot.backup.*
```

### 8.2 Main Configuration
```bash
# Edit main configuration file
sudo nano /etc/dovecot/dovecot.conf
```

Add/modify these settings:
```plaintext
# Basic Settings
protocols = imap lmtp
listen = *, ::

# Logging
log_path = /var/log/dovecot.log
info_log_path = /var/log/dovecot-info.log
debug_log_path = /var/log/dovecot-debug.log

# Mail Location
mail_location = maildir:/var/mail/vhosts/%d/%n

# System user used to access mails
default_internal_user = vmail
default_internal_group = vmail
```

### 8.3 SSL Configuration
```bash
# Edit SSL settings
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

Configure SSL settings:
```plaintext
# SSL/TLS Settings
ssl = required
ssl_cert = </etc/letsencrypt/live/mail.naphtal.tech/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.naphtal.tech/privkey.pem
ssl_min_protocol = TLSv1.2
ssl_prefer_server_ciphers = yes

# Cipher configuration
ssl_cipher_list = EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
ssl_dh_parameters_length = 2048
```

### 8.4 Authentication Configuration
```bash
# Edit authentication settings
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

Replace the configuration content with the following:
```plaintext
##
## Authentication processes
##

# Disable cleartext authentication unless using SSL/TLS
disable_plaintext_auth = yes

# Authentication mechanisms
auth_mechanisms = plain login

# User database
!include auth-system.conf.ext

# Authentication for Postfix
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}

# Default password scheme - this should be in the password database section
passdb {
  driver = passwd-file
  args = scheme=PLAIN username_format=%u /etc/dovecot/users
}

# User database
userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
}
```

### 8.5 Mail Location Configuration
```bash
# Edit mail location settings
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

Configure mail settings:
```plaintext
##
## Mail location and mailbox settings
##

# Mail location
mail_location = maildir:/var/mail/vhosts/%d/%n

# Mailbox settings
mail_privileged_group = mail
mail_access_groups = mail

# Mailbox home
mail_home = /var/mail/vhosts/%d/%n
```

### 8.6 Master Configuration
```bash
# Edit master configuration
sudo nano /etc/dovecot/conf.d/10-master.conf
```

Configure service settings:
```plaintext
# IMAP process
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

# LMTP process
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

### 8.7 Create Required Directories
```bash
# Create index directories
for user in admin user1 user2 user3 user4; do
    sudo mkdir -p "/var/mail/vhosts/naphtal.tech/$user/indexes"
    sudo chown -R vmail:vmail "/var/mail/vhosts/naphtal.tech/$user"
    sudo chmod -R 700 "/var/mail/vhosts/naphtal.tech/$user"
done

# Create log files
sudo touch /var/log/dovecot.log /var/log/dovecot-info.log /var/log/dovecot-debug.log
sudo chown vmail:vmail /var/log/dovecot*
```

### 8.8 Test Configuration
```bash
# Test Dovecot configuration
sudo dovecot -n

# Check for any syntax errors
doveconf -n

# Restart Dovecot
sudo systemctl restart dovecot

# Check service status
sudo systemctl status dovecot
```

Common Issues and Solutions:

1. Permission Issues:
```bash
# Fix common permission problems
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 700 /var/mail/vhosts
sudo chown -R vmail:dovecot /etc/dovecot
```

2. SSL Certificate Issues:
```bash
# Verify certificate permissions
sudo ls -l /etc/letsencrypt/live/mail.naphtal.tech/
sudo chmod -R 755 /etc/letsencrypt/live/
sudo chmod -R 755 /etc/letsencrypt/archive/
```

3. Log File Issues:
```bash
# Check log files for errors
sudo tail -f /var/log/dovecot.log
sudo journalctl -u dovecot
```

<a name="security-hardening"></a>
## 9. Security Hardening 🛡️

### 9.1 Fail2ban Configuration

1. Install and Configure Fail2ban:
```bash
# Install fail2ban if not already installed
sudo apt install fail2ban -y

# Create local configuration
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

2. Add Email Service Protection:
```plaintext
[postfix]
enabled = true
port = smtp,465,587
filter = postfix
logpath = /var/log/mail.log
maxretry = 5
findtime = 600
bantime = 3600

[dovecot]
enabled = true
port = imap,imaps
filter = dovecot
logpath = /var/log/mail.log
maxretry = 5
findtime = 600
bantime = 3600

[postfix-sasl]
enabled = true
port = smtp,465,587
filter = postfix-sasl
logpath = /var/log/mail.log
maxretry = 3
findtime = 600
bantime = 3600
```

3. Create Custom Filter:
```bash
sudo nano /etc/fail2ban/filter.d/postfix-custom.conf
```

Add custom filter rules:
```plaintext
[Definition]
failregex = ^%(__prefix_line)swarning: .*\[<HOST>\]: SASL (LOGIN|PLAIN) authentication failed
            ^%(__prefix_line)serror: .*\[<HOST>\]: SASL (LOGIN|PLAIN) authentication failed
ignoreregex =
```

### 9.2 Firewall Configuration (UFW)

```bash
# Install UFW if not present
sudo apt install ufw -y

# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (adjust port if different)
sudo ufw allow 22/tcp

# Allow mail ports
sudo ufw allow 25/tcp
sudo ufw allow 465/tcp
sudo ufw allow 587/tcp
sudo ufw allow 143/tcp
sudo ufw allow 993/tcp

# Allow HTTP/HTTPS for Let's Encrypt
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable

# Check status
sudo ufw status verbose
```

### 9.3 SSL/TLS Hardening

1. Update Postfix SSL Configuration:
```bash
sudo nano /etc/postfix/main.cf
```

Add/modify these settings:
```plaintext
# Strong SSL configuration
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes
smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_mandatory_ciphers = high
tls_high_cipherlist = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
```

2. Update Dovecot SSL Configuration:
```bash
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

Add/modify:
```plaintext
ssl_min_protocol = TLSv1.2
ssl_cipher_list = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384
ssl_prefer_server_ciphers = yes
```

### 9.4 System Hardening

1. Secure Shared Memory:
```bash
# Add to /etc/fstab
echo "tmpfs     /run/shm     tmpfs     defaults,noexec,nosuid     0     0" | sudo tee -a /etc/fstab
```

2. Update System Security Settings:
```bash
sudo nano /etc/sysctl.conf
```

Add security parameters:
```plaintext
# Network security
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.all.secure_redirects=0
net.ipv4.conf.default.secure_redirects=0
net.ipv4.conf.all.accept_source_route=0
net.ipv4.conf.default.accept_source_route=0

# Kernel hardening
kernel.sysrq=0
kernel.core_uses_pid=1
kernel.randomize_va_space=2
```

3. Apply changes:
```bash
sudo sysctl -p
```

### 9.5 Regular Security Maintenance

1. Create Security Maintenance Script:
```bash
sudo nano /usr/local/bin/security-check.sh
```

Add script content:
```bash
#!/bin/bash

# Update system
apt update && apt upgrade -y

# Check for failed login attempts
echo "=== Failed Login Attempts ==="
grep "authentication failed" /var/log/auth.log | tail -n 10

# Check Fail2ban status
echo -e "\n=== Fail2ban Status ==="
fail2ban-client status

# Check mail logs for suspicious activity
echo -e "\n=== Suspicious Mail Activity ==="
grep "authentication failed" /var/log/mail.log | tail -n 10

# Check SSL certificate expiry
echo -e "\n=== SSL Certificate Status ==="
certbot certificates

# Check disk usage
echo -e "\n=== Disk Usage ==="
df -h

# Check running services
echo -e "\n=== Service Status ==="
systemctl status postfix dovecot fail2ban | grep Active
```

Make script executable:
```bash
sudo chmod +x /usr/local/bin/security-check.sh
```

2. Set Up Automatic Security Updates:
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

## 10. Testing & Verification 🧪

### 10.1 Service Status Verification
```bash
# Check all service statuses
for service in postfix dovecot mysql; do
    echo "=== Checking $service status ==="
    sudo systemctl status $service | grep -E "Active:|loaded"
    echo "----------------------------------------"
done

# Verify running processes
sudo netstat -tlpn | grep -E ':25|:465|:587|:143|:993'
```

Expected Output:
```plaintext
=== Checking postfix status ===
   Loaded: loaded (/lib/systemd/system/postfix.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2024-03-20 12:00:00 UTC; 1h ago
----------------------------------------
tcp        0      0 0.0.0.0:25            0.0.0.0:*               LISTEN      1234/master
tcp        0      0 0.0.0.0:465           0.0.0.0:*               LISTEN      1234/master
tcp        0      0 0.0.0.0:587           0.0.0.0:*               LISTEN      1234/master
tcp        0      0 0.0.0.0:143           0.0.0.0:*               LISTEN      5678/dovecot
tcp        0      0 0.0.0.0:993           0.0.0.0:*               LISTEN      5678/dovecot
```

### 10.2 Test SMTP Connection
```bash
# Test SMTP connection (port 25)
nc -v mail.naphtal.tech 25

# Expected interaction:
# > EHLO mail.naphtal.tech
# > QUIT
```

### 10.3 Test IMAP Connection
```bash
# Test IMAP connection (port 143)
nc -v mail.naphtal.tech 143

# Expected interaction:
# > a LOGIN "user1@naphtal.tech" "password"
# > a LIST "" "*"
# > a LOGOUT
```

### 10.4 SSL Certificate Verification
```bash
# Test SSL on IMAPS (993)
openssl s_client -connect mail.naphtal.tech:993 -showcerts

# Test SSL on SMTPS (465)
openssl s_client -connect mail.naphtal.tech:465 -showcerts

# Verify certificate validity
echo | openssl s_client -connect mail.naphtal.tech:993 2>/dev/null | \
    openssl x509 -noout -text | grep "Not After"
```

### 10.5 Send Test Emails
```bash
# Install mail command if not present
sudo apt install mailutils -y

# Send test email internally
echo "Test email body" | mail -s "Test Subject" user1@naphtal.tech

# Check mail logs
sudo tail -f /var/log/mail.log
```

### 10.6 Test Authentication
```bash
# Test SMTP Authentication
printf "AUTH LOGIN\n$(echo -n "user1@naphtal.tech" | base64)\n$(echo -n "password" | base64)\n" | \
openssl s_client -connect mail.naphtal.tech:587 -starttls smtp -quiet
```

### 10.7 DNS Record Verification
```bash
# Comprehensive DNS check script
for record in A MX TXT SPF DMARC; do
    echo "=== Checking $record records ==="
    case $record in
        "A")
            dig +short mail.naphtal.tech A
            ;;
        "MX")
            dig +short naphtal.tech MX
            ;;
        "TXT"|"SPF")
            dig +short naphtal.tech TXT
            ;;
        "DMARC")
            dig +short _dmarc.naphtal.tech TXT
            ;;
    esac
    echo "----------------------------------------"
done
```

### 10.8 Mail Directory Structure Check
```bash
# Check mail directory permissions and structure
sudo tree -pug /var/mail/vhosts/naphtal.tech/

# Verify user mailbox access
for user in admin user1 user2 user3 user4; do
    echo "=== Checking $user mailbox ==="
    sudo ls -la /var/mail/vhosts/naphtal.tech/$user
    echo "----------------------------------------"
done
```

### 10.9 Log Analysis
```bash
# Create log analysis script
cat << 'EOF' > check_mail_logs.sh
#!/bin/bash

echo "=== Last 10 Authentication Attempts ==="
grep "auth" /var/log/mail.log | tail -n 10

echo -e "\n=== Recent Connection Attempts ==="
grep "connect from" /var/log/mail.log | tail -n 5

echo -e "\n=== Recent Email Deliveries ==="
grep "status=sent" /var/log/mail.log | tail -n 5

echo -e "\n=== Recent Errors ==="
grep "error" /var/log/mail.log | tail -n 5
EOF

chmod +x check_mail_logs.sh
sudo ./check_mail_logs.sh
```

Common Issues and Solutions:

1. Connection Refused:
```bash
# Check if services are running
sudo systemctl restart postfix dovecot

# Verify firewall settings
sudo ufw status
```

2. Authentication Failures:
```bash
# Check password file permissions
sudo ls -l /etc/dovecot/users
sudo chown vmail:dovecot /etc/dovecot/users
sudo chmod 640 /etc/dovecot/users
```

3. Mail Delivery Issues:
```bash
# Check mail queue
sudo postqueue -p

# Attempt to flush queue
sudo postqueue -f

# Check for blocked ports
sudo iptables -L
```

<a name="client-setup"></a>
## 11. Email Client Setup 📱

### 11.1 General Settings for All Clients

#### Incoming Mail (IMAP) Settings:
```plaintext
Server: mail.naphtal.tech
Port: 993 (SSL/TLS) or 143 (STARTTLS)
Username: full email (e.g., user1@naphtal.tech)
Password: [your password]
Security: SSL/TLS or STARTTLS
Authentication: Normal Password
```

#### Outgoing Mail (SMTP) Settings:
```plaintext
Server: mail.naphtal.tech
Port: 587 (STARTTLS) or 465 (SSL/TLS)
Username: full email (same as incoming)
Password: [your password]
Security: STARTTLS or SSL/TLS
Authentication: Normal Password
```

### 11.2 Thunderbird Setup

1. Account Creation:
```plaintext
1. Click Menu ≡ → New → Email Account
2. Enter:
   - Your name: [User's Name]
   - Email address: user1@naphtal.tech
   - Password: [your password]
3. Click 'Configure manually'
4. Enter server settings as above
```

2. Advanced Settings:
```plaintext
1. Server Settings → Server Settings:
   - Connection security: SSL/TLS
   - Authentication method: Normal password
   
2. Outgoing Server (SMTP):
   - Connection security: STARTTLS
   - Authentication method: Normal password
   - Server name: mail.naphtal.tech
```

### 11.3 Outlook Setup

1. Account Addition:
```plaintext
1. File → Add Account
2. Choose 'Manual setup or additional server types'
3. Choose 'POP or IMAP'
4. Enter:
   - Your Name: [User's Name]
   - Email Address: user1@naphtal.tech
   - Account Type: IMAP
   - Incoming mail server: mail.naphtal.tech
   - Outgoing mail server: mail.naphtal.tech
```

2. Advanced Settings:
```plaintext
1. More Settings → Advanced:
   Incoming Server (IMAP):
   - Port: 993
   - Encryption: SSL/TLS

   Outgoing Server (SMTP):
   - Port: 587
   - Encryption: STARTTLS
```

### 11.4 iPhone/iPad Setup

1. iOS Mail App:
```plaintext
1. Settings → Mail → Accounts → Add Account
2. Choose 'Other' → Add Mail Account
3. Enter:
   - Name: [User's Name]
   - Email: user1@naphtal.tech
   - Password: [your password]
   - Description: Naphtal Tech Mail

4. Choose 'IMAP' and enter server information:
   Incoming Mail Server:
   - Host Name: mail.naphtal.tech
   - Username: user1@naphtal.tech
   - Password: [your password]

   Outgoing Mail Server:
   - Host Name: mail.naphtal.tech
   - Username: user1@naphtal.tech
   - Password: [your password]
```

### 11.5 Android Setup

1. Gmail App:
```plaintext
1. Open Gmail → Menu → Settings → Add account
2. Choose 'Other'
3. Enter email address: user1@naphtal.tech
4. Choose 'Personal (IMAP)'
5. Enter password
6. Enter server settings:
   
   Incoming Server:
   - Server: mail.naphtal.tech
   - Port: 993
   - Security type: SSL/TLS
   
   Outgoing Server:
   - Server: mail.naphtal.tech
   - Port: 587
   - Security type: STARTTLS
```

### 11.6 Testing Email Client Setup

1. Send Test Email:
```plaintext
1. Compose new email
2. To: admin@naphtal.tech
3. Subject: Test Email from [Client Name]
4. Body: This is a test email from [Client Name]
5. Send and verify delivery
```

2. Verify Settings:
```plaintext
1. Check Sent folder
2. Check server logs:
   sudo tail -f /var/log/mail.log
3. Verify received email
4. Test reply functionality
```

Common Issues and Solutions:

1. Connection Errors:
```plaintext
Problem: "Unable to connect to server"
Solution: 
- Verify server address is correct
- Check ports are not blocked
- Ensure SSL/TLS settings match
```

2. Authentication Failures:
```plaintext
Problem: "Invalid username or password"
Solution:
- Verify full email address is used as username
- Check password is correct
- Ensure authentication method matches server
```

3. SSL Certificate Warnings:
```plaintext
Problem: "Certificate not trusted"
Solution:
- Verify certificate is valid
- Check certificate is properly installed
- Update client's trusted certificates
```

<a name="monitoring"></a>
## 12. Monitoring & Maintenance 📊

### 12.1 Setup Monitoring Tools
```bash
# Install monitoring packages
sudo apt install monitoring-plugins logwatch -y

# Create monitoring directory
sudo mkdir -p /opt/monitoring
```

### 12.2 Email Server Monitoring Script
```bash
sudo nano /opt/monitoring/mail_monitor.sh
```

Add script content:
```bash
#!/bin/bash

LOG_FILE="/var/log/mail_monitor.log"
ADMIN_EMAIL="admin@naphtal.tech"

# Check services
check_services() {
    for service in postfix dovecot mysql; do
        if ! systemctl is-active --quiet $service; then
            echo "[$(date)] ERROR: $service is down!" >> $LOG_FILE
            echo "$service is down!" | mail -s "Service Alert: $service" $ADMIN_EMAIL
        fi
    done
}

# Check disk space
check_disk() {
    DISK_USAGE=$(df -h | grep '/dev/xvda1' | awk '{print $5}' | cut -d'%' -f1)
    if [ "$DISK_USAGE" -gt 80 ]; then
        echo "[$(date)] WARNING: Disk usage at ${DISK_USAGE}%" >> $LOG_FILE
        echo "Disk usage warning: ${DISK_USAGE}%" | mail -s "Disk Space Alert" $ADMIN_EMAIL
    fi
}

# Main execution
check_services
check_disk
```

### 12.3 Setup Automated Maintenance
```bash
# Make script executable
sudo chmod +x /opt/monitoring/mail_monitor.sh

# Add to crontab
sudo crontab -e
```

Add these lines:
```plaintext
# Run monitoring every 5 minutes
*/5 * * * * /opt/monitoring/mail_monitor.sh

# Daily log rotation at 2 AM
0 2 * * * /usr/sbin/logrotate /etc/logrotate.d/mail
```

<a name="django-flask"></a>
## 13. Django/Flask Email Integration 🐍

### 13.1 Django Email Configuration

Add to your Django settings.py:
```python
# Email Configuration
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'mail.naphtal.tech'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
EMAIL_HOST_USER = 'your-email@naphtal.tech'
EMAIL_HOST_PASSWORD = 'your-email-password'
DEFAULT_FROM_EMAIL = 'your-email@naphtal.tech'
```

Example Django view for sending emails:
```python
from django.core.mail import send_mail
from django.conf import settings

def send_email_view(request):
    try:
        send_mail(
            subject='Test Email from Django',
            message='This is a test email from your Django application.',
            from_email=settings.DEFAULT_FROM_EMAIL,
            recipient_list=['recipient@example.com'],
            fail_silently=False,
        )
        return HttpResponse("Email sent successfully!")
    except Exception as e:
        return HttpResponse(f"Failed to send email: {str(e)}")
```

### 13.2 Flask Email Configuration

Install required package:
```bash
pip install flask-mail
```

Flask application configuration:
```python
from flask import Flask
from flask_mail import Mail, Message

app = Flask(__name__)

# Mail Server Configuration
app.config['MAIL_SERVER'] = 'mail.naphtal.tech'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = 'your-email@naphtal.tech'
app.config['MAIL_PASSWORD'] = 'your-email-password'
app.config['MAIL_DEFAULT_SENDER'] = 'your-email@naphtal.tech'

mail = Mail(app)

@app.route('/send-email')
def send_email():
    try:
        msg = Message(
            subject='Test Email from Flask',
            recipients=['recipient@example.com'],
            body='This is a test email from your Flask application.'
        )
        mail.send(msg)
        return "Email sent successfully!"
    except Exception as e:
        return f"Failed to send email: {str(e)}"

if __name__ == '__main__':
    app.run(debug=True)
```

### 13.3 Testing Email Integration

1. Django Test Script:
```python
# test_email.py
import django
django.setup()

from django.core.mail import send_mail
from django.conf import settings

def test_email_configuration():
    try:
        send_mail(
            'Test Email',
            'This is a test email.',
            settings.DEFAULT_FROM_EMAIL,
            ['test@example.com'],
            fail_silently=False,
        )
        print("Test email sent successfully!")
    except Exception as e:
        print(f"Error sending email: {e}")

if __name__ == "__main__":
    test_email_configuration()
```

2. Flask Test Script:
```python
# test_flask_email.py
from your_flask_app import app, mail
from flask_mail import Message

def test_flask_email():
    with app.app_context():
        try:
            msg = Message(
                'Test Flask Email',
                recipients=['test@example.com']
            )
            msg.body = "This is a test email from Flask."
            mail.send(msg)
            print("Test email sent successfully!")
        except Exception as e:
            print(f"Error sending email: {e}")

if __name__ == "__main__":
    test_flask_email()
```

### 13.4 Error Handling and Logging

Add this to your Django/Flask application:
```python
import logging

# Configure logging
logging.basicConfig(
    filename='email_logs.log',
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def send_email_with_logging(subject, message, recipient_list):
    try:
        # Django
        send_mail(subject, message, settings.DEFAULT_FROM_EMAIL, recipient_list)
        logging.info(f"Email sent successfully to {recipient_list}")
    except Exception as e:
        logging.error(f"Failed to send email: {str(e)}")
        raise
```

<a name="troubleshooting"></a>
## 14. Troubleshooting Guide 🔧

### 14.1 Common Issues and Solutions

1. Connection Refused:
```bash
# Check if services are running
sudo systemctl restart postfix dovecot

# Verify firewall settings
sudo ufw status
```

2. Authentication Failures:
```bash
# Verify user permissions
sudo postconf -e 'smtpd_sasl_auth_enable = yes'
sudo postconf -e 'broken_sasl_auth_clients = yes'

# Restart services
sudo systemctl restart postfix
sudo systemctl restart dovecot
```

3. SSL Certificate Warnings:
```bash
# Verify certificate permissions
sudo ls -l /etc/letsencrypt/live/mail.naphtal.tech/
sudo chmod -R 755 /etc/letsencrypt/live/
sudo chmod -R 755 /etc/letsencrypt/archive/
```

4. Mail Delivery Issues:
```bash
# Check mail queue
sudo postqueue -p

# Attempt to flush queue
sudo postqueue -f

# Check for blocked ports
sudo iptables -L
```

### 14.2 Additional Resources

1. Postfix Documentation:
   - [Postfix Documentation](https://www.postfix.org/documentation.html)

2. Dovecot Documentation:
   - [Dovecot Documentation](https://www.dovecot.org/documentation.html)

3. Certbot Documentation:
   - [Certbot Documentation](https://certbot.eff.org/docs/)

4. Fail2ban Documentation:
   - [Fail2ban Documentation](https://www.fail2ban.org/wiki/index.php/Main_Page)

5. UFW Documentation:
   - [UFW Documentation](https://wiki.ubuntu.com/UncomplicatedFirewall)

6. System Security Documentation:
   - [System Security Documentation](https://www.debian.org/doc/manuals/securing-debian-howto/)

7. Monitoring Documentation:
   - [Monitoring Documentation](https://www.monitoring-plugins.org/doc/)

8. Django Email Integration Documentation:
   - [Django Email Integration Documentation](https://docs.djangoproject.com/en/stable/topics/email/)

9. Flask Email Integration Documentation:
   - [Flask Email Integration Documentation](https://flask-mail.readthedocs.io/)

10. Security Hardening Documentation:
    - [Security Hardening Documentation](https://www.debian.org/doc/manuals/securing-debian-howto/)

