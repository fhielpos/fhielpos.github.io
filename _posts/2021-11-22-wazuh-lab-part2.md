---
layout: post
title: "Setting up a Wazuh lab - Part 2 [Public SSL certificates]"
---

Following on my Wazuh lab deployment, this post aims to install Public SSL certificates for the web interface with Let's Encrypt.

<img src="assets/img/letsencrypt.png" alt="Let's Encrypt" width=500px height=auto>

To avoid the "*Your connection is not private*" warning for using **Self Signed Certificates**, we need to use a trusted `Certificate Authority`. **Let's Encrypt** is a free, automated, and open certificate authority which is great for our use case.

## DNS record

In order to use a **wildcard certificate**, we will need a domain. I already have this one, so I will add a subdomain to my `DNS server` in **Google Cloud**.

In my DNS console, I will add a `type A` DNS record pointing `wazuh.fhielpos.ar` to my Droplet IP.

## Creating the certificate

The easiest way to create the certificates is with `certbot`. Let's install it:

```bash
# Add certbot repo
sudo add-apt-repository ppa:certbot/certbot
# Update
sudo apt-get update
# Install certbot
sudo apt-get install python-certbot-apache
```

Then, we can create our certs with:
```bash
certbot certonly --standalone -d wazuh.fhielpos.ar
```

The previous will create our certificates inside `/etc/letsencrypt/live/wazuh.fhielpos.ar`

## Copying the certificates to Kibana

Now that we have our certificates, we need to move them to the Kibana directory:

```bash
cd /etc/letsencrypt/live/wazuh.fhielpos.ar/
mv cert.pem /etc/kibana/certs/https.pem
mv privkey.pem /etc/kibana/certs/https-key.pem
```

Now that we have our certificates, we also need the `Certificate Authority` so Kibana can trust it. We can download it from here:

* [Let's Encrypt - Certificate Authority](https://letsencrypt.org/certificates/#root-certificates)

In my case, I will download the following:

```bash
curl -so /etc/kibana/certs/https-ca.pem https://letsencrypt.org/certs/isrgrootx1.pem
```

Finally, we will make sure that these certificates are accessible by Kibana by changing the ownership:

```bash
chown kibana:kibana -R /etc/kibana/certs
```

## Configuring Kibana to use the certificates

Finally is time to configure Kibana to use our brand new certificates. We need to modify the `/etc/kibana/kibana.yml` and set the following settings:

```yaml
server.ssl.enabled: true
server.ssl.key: "/etc/kibana/certs/https.pem"
server.ssl.certificate: "/etc/kibana/certs/https-key.pem"
server.ssl.certificateAuthorities: ["/etc/kibana/certs/https-ca.pem"]
```

Once done, simply restart the Kibana service and access your domain.

```bash
systemctl restart kibana
```


