---
title: "SSL Certificate"
tags: 
- SSL
- domain
- certificate
- encryption
---

## Let's Encrypt

Let's Encrypt provide free SSL certificate:
- They even support wildcard certificate!
- Certificate valid for 90 days.
- You could obtain or renew certificates automatically or manually.

### Obtain Certificate

Suppose:
- Your domain is someDomain.com
- You want to obtain certificate for address abc.someDomain.com and xyz.someDomain.com
- You email address is yourEmail@someDomain.com (this is important cause Let's Encrypt will send nortification emails before certificate expire)
- OS is Debian

Install software:
```bash
sudo apt update
sudo apt install certbot
```

Obtain certificate:
```
sudo certbot --manual --agree-tos --hsts --staple-ocsp --email yourEmail@someDomain.com -d abc.someDomain.com -d xyz.someDomain.com --rsa-key-size 4096  --preferred-challenges dns certonly

# The certbot software will guide you step by step.
# Will ask you to add 2 TEXT records to domain DNS.
# Needs 2 records because we request a certificate for 2 addresses.
# You can delete these 2 TEXT records after obtain certificate obtain successfully.
# certbot will tell you if succeed and where is the generated certificates.
```
