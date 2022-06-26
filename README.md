# acme-tiny

[![Tests](https://github.com/diafygi/acme-tiny/actions/workflows/full-tests-with-coverage.yml/badge.svg)](https://github.com/diafygi/acme-tiny/actions/workflows/full-tests-with-coverage.yml)
[![Coverage Status](https://coveralls.io/repos/github/diafygi/acme-tiny/badge.svg?branch=master)](https://coveralls.io/github/diafygi/acme-tiny?branch=master)

** 免费申请ssl证书的脚本很多，一眼能看到代码干净安全的就这个20220622 **
This is a tiny, auditable script that you can throw on your server to issue
and renew [Let's Encrypt](https://letsencrypt.org/) certificates. Since it has
to be run on your server and have access to your private Let's Encrypt account
key, I tried to make it as tiny as possible (currently less than 200 lines).
The only prerequisites are python and openssl.

**PLEASE READ THE SOURCE CODE! YOU MUST TRUST IT WITH YOUR PRIVATE ACCOUNT KEY!**

## Donate

If this script is useful to you, please donate to the EFF. I don't work there,
but they do fantastic work.

[https://eff.org/donate/](https://eff.org/donate/)

## How to use this script

If you already have a Let's Encrypt issued certificate and just want to renew,
you should only have to do Steps 3 and 6.

### Step 1: Create a Let's Encrypt account private key (if you haven't already)

You must have a public key registered with Let's Encrypt and sign your requests
with the corresponding private key. If you don't understand what I just said,
this script likely isn't for you! Please use the official Let's Encrypt
[client](https://github.com/letsencrypt/letsencrypt).
To accomplish this you need to initially create a key, that can be used by
acme-tiny, to register an account for you and sign all following requests.

```
openssl genrsa 4096 > account.key
```

#### Use existing Let's Encrypt key

Alternatively you can convert your key, previously generated by the original
Let's Encrypt client.

The private account key from the Let's Encrypt client is saved in the
[JWK](https://tools.ietf.org/html/rfc7517) format. `acme-tiny` is using the PEM
key format. To convert the key, you can use the tool
[conversion script](https://gist.github.com/JonLundy/f25c99ee0770e19dc595) by JonLundy:

```sh
# Download the script
wget -O - "https://gist.githubusercontent.com/JonLundy/f25c99ee0770e19dc595/raw/6035c1c8938fae85810de6aad1ecf6e2db663e26/conv.py" > conv.py

# Copy your private key to your working directory
cp /etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/<id>/private_key.json private_key.json

# Create a DER encoded private key
openssl asn1parse -noout -out private_key.der -genconf <(python2 conv.py private_key.json)

# Convert to PEM
openssl rsa -in private_key.der -inform der > account.key
```

### Step 2: Create a certificate signing request (CSR) for your domains.

The ACME protocol (what Let's Encrypt uses) requires a CSR file to be submitted
to it, even for renewals. You can use the same CSR for multiple renewals. NOTE:
you can't use your account private key as your domain private key!

```
# Generate a domain private key (if you haven't already)
openssl genrsa 4096 > domain.key
```

```
# For a single domain
openssl req -new -sha256 -key domain.key -subj "/CN=yoursite.com" > domain.csr

# For multiple domains (use this one if you want both www.yoursite.com and yoursite.com)
openssl req -new -sha256 -key domain.key -subj "/" -addext "subjectAltName = DNS:yoursite.com, DNS:www.yoursite.com" > domain.csr

# For multiple domains (same as above but works with openssl < 1.1.1)
openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:yoursite.com,DNS:www.yoursite.com")) > domain.csr
```

### Step 3: Make your website host challenge files

You must prove you own the domains you want a certificate for, so Let's Encrypt
requires you host some files on them. This script will generate and write those
files in the folder you specify, so all you need to do is make sure that this
folder is served under the ".well-known/acme-challenge/" url path. NOTE: Let's
Encrypt will perform a plain HTTP request to port 80 on your server, so you
must serve the challenge files via HTTP (a redirect to HTTPS is fine too).

```
# Make some challenge folder (modify to suit your needs)
mkdir -p /var/www/challenges/
```

```nginx
# Example for nginx
server {
    listen 80;
    server_name yoursite.com www.yoursite.com;

    location /.well-known/acme-challenge/ {
        alias /var/www/challenges/;
        try_files $uri =404;
    }

    ...the rest of your config
}
```

### Step 4: Get a signed certificate!

Now that you have setup your server and generated all the needed files, run this
script on your server with the permissions needed to write to the above folder
and read your private account key and CSR.

```
# Run the script on your server
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /var/www/challenges/ > ./signed_chain.crt
```

### Step 5: Install the certificate

The signed https certificate chain that is output by this script can be used along
with your private key to run an https server. You need to include them in the
https settings in your web server's configuration. Here's an example on how to
configure an nginx server:

```nginx
server {
    listen 443 ssl;
    server_name yoursite.com www.yoursite.com;

    ssl_certificate /path/to/signed_chain.crt;
    ssl_certificate_key /path/to/domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_session_cache shared:SSL:50m;
    ssl_dhparam /path/to/server.dhparam;
    ssl_prefer_server_ciphers on;

    ...the rest of your config
}

server {
    listen 80;
    server_name yoursite.com www.yoursite.com;

    location /.well-known/acme-challenge/ {
        alias /var/www/challenges/;
        try_files $uri =404;
    }

    ...the rest of your config
}
```

### Step 6: Setup an auto-renew cronjob

Congrats! Your website is now using https! Unfortunately, Let's Encrypt
certificates only last for 90 days, so you need to renew them often. No worries!
It's automated! Just make a bash script and add it to your crontab (see below
for example script).

Example of a `renew_cert.sh`:
```sh
#!/usr/bin/sh
python /path/to/acme_tiny.py --account-key /path/to/account.key --csr /path/to/domain.csr --acme-dir /var/www/challenges/ > /path/to/signed_chain.crt.tmp || exit
mv /path/to/signed_chain.crt.tmp /path/to/signed_chain.crt
service nginx reload
```

```
# Example line in your crontab (runs once per month)
0 0 1 * * /path/to/renew_cert.sh 2>> /var/log/acme_tiny.log
```

**NOTE:** Since Let's Encrypt's ACME v2 release (acme-tiny 4.0.0+), the intermediate
certificate is included in the issued certificate download, so you no longer have
to independently download the intermediate certificate and concatenate it to your
signed certificate. If you have an shell script or Makefile using acme-tiny &lt;4.0 (e.g. before
2018-03-17) with acme-tiny 4.0.0+, then you may be adding the intermediate
certificate to your signed_chain.crt twice (which
[causes issues with at least GnuTLS 3.7.0](https://gitlab.com/gnutls/gnutls/-/issues/1131)
besides making the certificate slightly larger than it needs to be). To fix,
simply remove the bash code where you're downloading the intermediate and adding
it to the acme-tiny certificate output.

## Permissions

The biggest problem you'll likely come across while setting up and running this
script is permissions. You want to limit access to your account private key and
challenge web folder as much as possible. I'd recommend creating a user
specifically for handling this script, the account private key, and the
challenge folder. Then add the ability for that user to write to your installed
certificate file (e.g. `/path/to/signed_chain.crt`) and reload your webserver. That
way, the cron script will do its thing, overwrite your old certificate, and
reload your webserver without having permission to do anything else.

**BE SURE TO:**
* Backup your account private key (e.g. `account.key`)
* Don't allow this script to be able to read your domain private key!
* Don't allow this script to be run as root!

## Staging Environment

Let's Encrypt recommends testing new configurations against their staging servers,
so when testing out your new setup, you can use
`--directory-url https://acme-staging-v02.api.letsencrypt.org/directory`
to issue fake test certificates instead of real ones from Let's Encrypt's production servers.
See [https://letsencrypt.org/docs/staging-environment/](https://letsencrypt.org/docs/staging-environment/)
for more details.

## Feedback/Contributing

This project has a very, very limited scope and codebase. I'm happy to receive
bug reports and pull requests, but please don't add any new features. This
script must stay under 200 lines of code to ensure it can be easily audited by
anyone who wants to run it.

If you want to add features for your own setup to make things easier for you,
please do! It's open source, so feel free to fork it and modify as necessary.
