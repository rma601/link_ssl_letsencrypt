# Securing Chainlink Nodes with Letsencrypt

This guide was written to assist node operators with getting letsencrypt certificates into their docker environment for securing their nodes. 

This guide follows [this guide](https://pentacent.medium.com/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71) to obtain the certificates, and [this guide](https://docs.chain.link/docs/enabling-https-connections/) to install them. Refer to the links provided if you need context that may not be provided here. 

### Assumptions:
- You should already have some DNS entry pointing to the IP of your node. You can use services such as [DynDNS](https://account.dyn.com/) to accomplish this goal
- We will not be covering steps to get your node up and running. Please follow the [official guidance](https://docs.chain.link/docs/running-a-chainlink-node/) before using this documentation.
- Steps outlined here will be related to a RHEL-based environment. You may need to translate your firewall commands accordingly
- I typically throw configurations in /opt, though you may want to put them elsewhere. 
- You will need to have `docker-compose` installed. 

## Obtaining Certificates Using Docker

As mentioned above, we will be using [this guide](https://pentacent.medium.com/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71) as a foundation for obtaining certificates. 

First, we need to configure our directory structure.

```
mkdir -p /opt/ssl
mkdir -p /opt/nginx
```

Then, we need to create our docker compose file. This should probably live in your home directory

```
version: '2.1'
services:
  nginx:
    image: nginx:1.15-alpine
    container_name: nginx
    command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /opt/nginx:/etc/nginx/conf.d
      - /opt/ssl/conf:/etc/letsencrypt
      - /opt/ssl/www:/var/www/certbot
  certbot:
    image: certbot/certbot
    container_name: certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
    volumes:
      - /opt/ssl/conf:/etc/letsencrypt
      - /opt/ssl/www:/var/www/certbot
```

Now, we set our nginx configuration. Replace the sections labeled `your.fqdn.org` with the fully qualified domain name for your server. 

```
touch /opt/nginx/app.conf
## edit app.conf above with your favorite text editor to add the following ##
server {
    listen 80;
    server_name your.fqdn.org;
    location / {
        return 301 https://$host$request_uri;
    }
}
server {
    listen 443 ssl;
    server_name your.fqdn.org;
    ssl_certificate /etc/letsencrypt/live/your.fqdn.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your.fqdn.org/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

location /.well-known/acme-challenge/ {
    root /var/www/certbot;
}


}

```

Finally, we  add the required firewall rules and run a shell script. This script places fake files to get nginx up and running, then replaces them with real files that we can use to grab a certificate. I have included a copy of the script in this github repo, for ease of access. 


```
## You need port 80 and 443 open in order to have your certificates validated by certbot and letsencrypt

firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

## edit the script with your favorite text editor, and set the following values ##

domains=(your.fqdn.org)
rsa_key_size=4096
data_path="/opt/ssl"
email="" # Adding a valid address is strongly recommended
staging=0 # Set to 1 if you're testing your setup to avoid hitting request limits

## Run the script ##
./get_certs.sh
```

If you do everything correctly, you should have certificates in the `/opt/ssl/conf/live/your.fqdn.org/` directory. These files are symlinks, however, so keep that in mind if you plan to mount these to your containers. Both the symlink and the actual path will need to be visible internally to the container environment. 

## Configuring Chainlink Nodes for SSL

Once you have your certificates in hand, you can follow the [official guidance](https://docs.chain.link/docs/enabling-https-connections/) to secure your nodes. Here is how I did it:

Edit the ENV files for your nodes to include the following:

```
## /opt/link/.env

## Add following tls lines
TLS_CERT_PATH=/ssl/conf/live/your.fqdn.org/cert.pem
TLS_KEY_PATH=/ssl/conf/live/your.fqdn.org/privkey.pem

## Remove the following lines disabling HTTPS
CHAINLINK_TLS_PORT=0
SECURE_COOKIES=false

## /opt/ocr/.env 

## Uncomment HTTPS settings
# Settings for HTTPS (enable these or the ones below for http)
CHAINLINK_TLS_PORT=6690
SECURE_COOKIES=true
TLS_CERT_PATH=/ssl/conf/live/your.fqdn.org/cert.pem
TLS_KEY_PATH=/ssl/conf/live/your.fqdn.org/privkey.pem

## Comment out HTTP settings
# Setting for HTTP
#CHAINLINK_TLS_PORT=0
#SECURE_COOKIES=false
```

Now, tear down and recreate your containers. You will need to add `-v /opt/ssl:/ssl` to your docker run commands, or `- /opt/ssl:/ssl` to the `volumes` sections of your docker compose file. You will also need to change your port mapping to reflect the ssl configuration for your nodes. By default, port `6689` is used for all HTTPS communications. If you are running an OCR node, you will need to configure a seperate port so that the two services do not conflict. I chose to use `6690` as my secondary port. 

You will likely also need to open and close the appropriate ports:

```
firewall-cmd --permanent --add-port=6689
firewall-cmd --permanent --add-port=6690
firewall-cmd --permanent --remove-port=6688
firewall-cmd --reload
```

Thats it. You have secured your containers using letsencrypt certificates. Go have a drink, or something. 