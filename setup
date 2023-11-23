#!/bin/bash

# stops processing in case of failure
set -euo pipefail

# prints each line executed
set -x

##################################################  VARS  ##################################################

netzh="aferrarini"
domain="$netzh.student.dtf.netsec.inf.ethz.ch"
acme_dir="https://acme.dtf.netsec.inf.ethz.ch/acme/default/directory"


################################################## TASK 1 ##################################################

# Remove line which blocks all traffic from conf file
sudo sed -i '/ct state established,related log prefix "DROPPING PACKET: " drop/d' /etc/nftables.conf

# Restart firewall
sudo systemctl restart nftables


################################################## TASK 2 ##################################################

# Load conf file to kernel and add new rule
sudo nft flush ruleset
sudo nft -f /etc/nftables.conf 
sudo nft add rule inet filter input ip saddr grader.dtf.netsec.inf.ethz.ch tcp dport {5432} accept
sudo nft add rule inet filter input tcp dport 5432 drop

# Dump conf file in kernel to conf file on disk
sudo chmod 777 /etc/nftables.conf
sudo nft list ruleset > /etc/nftables.conf
sudo chmod 755 /etc/nftables.conf

# Restart firewall
sudo systemctl restart nftables


################################################## TASK 3 ##################################################

# Change permissions on unencrypted passwords file
sudo chmod 600 /var/www/secret/passwords


################################################## TASK 4 ##################################################

# 0 - Download ACME client
rm -rf ../req_cert
mkdir ../req_cert
cd ../req_cert
git clone https://github.com/diafygi/acme-tiny.git

# 1 - Generate private key
sudo openssl genrsa 4096 > account.key

# 2 - Generate domain private key and CSR from the generated key
sudo openssl genrsa 4096 > domain.key
sudo openssl req -new -sha256 -key domain.key -subj "/CN=$domain" > domain.csr

# 3 - Serve challenge files
sudo mkdir -p /var/www/html/.well-known/acme-challenge

# 4 - Request certificate signing
sudo python3 ./acme-tiny/acme_tiny.py --account-key ./account.key\
	--csr ./domain.csr\
	--acme-dir /var/www/html/.well-known/acme-challenge\
	--directory-url $acme_dir\
	--disable-check	> ./signed_chain.crt

# 5 - Configuring certificate on NGINX
sudo sed -i "/gzip on;/s/$/\n\tserver {\n\t\tlisten 443;\n\t\tssl on;\n\t\tserver_name aferrarini.student.dtf.netsec.inf.ethz.ch;\n\t\tssl_certificate \/home\/aferrarini\/req_cert\/signed_chain.crt;\n\t\tssl_certificate_key \/home\/aferrarini\/req_cert\/domain.key;\n\t\tlocation \/ {\n\t\t\troot \/var\/www\/html;\n\t\t\tindex index.html;\n\t\t}\n\t}/" /etc/nginx/nginx.conf

# 6 - Finally, restarting NGINX
sudo systemctl restart nginx


################################################## TASK 5 ##################################################

sudo sed -i "s/ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE/ssl_protocols TLSv1.3;/" /etc/nginx/nginx.conf
sudo systemctl restart nginx

