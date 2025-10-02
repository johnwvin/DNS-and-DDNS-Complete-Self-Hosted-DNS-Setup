# Technitium DNS and DDNS Complete Self-Hosted DNS Setup Guide

A simple, easy-to-follow guide on setting up your own DNS Server with Technitium, and cloudflare's DDNS server to update your public DNS records when your public IP address dynamically changes.

Prerequisites:
- A registered domain name
- A Cloudflare account with your public domain onboarded (this is easy to achieve, simply make an account and click the onboard button then follow the steps)
- A server/VM running Ubuntu 24

Why Technitium?
- Technitium is a reliable and simple DNS server, equipped with a GUI for ease of management.
  
Why DDNS?
- We need a DDNS server if our public IP assignment is dynamic, if your's is static don't worry about setting up DDNS, simply create your A records on Cloudflare's website statically and you're done. You don't have to worry about your public IP changes being reflected on your public DNS records.

# Technitium DNS Server and Cloudflare DDNS installation:

First, access your Ubuntu Server update, upgrade, and install docker and docker-compose with their brief online guide: https://docs.docker.com/engine/install/ubuntu/

Next, disable the local DNS server already running on the ubuntu machine:

```
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo rm /etc/resolv.conf
echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" | sudo tee /etc/resolv.conf # change this command to match your preferred dns servers
```

Create these directories to use:

```
mkdir ~/network-services
mkdir ~/network-services/dns
mkdir ~/network-services/ddns
```

## Technitium

Create the Technitium DNS server docker-compose .yaml file
```
nano ~/network-services/docker-compose.yml
```

Insert this into the docker-compose.yml file and replace the placeholders with your info:

```
version: "3.7"

services:
  dns-server:
    image: technitium/dns-server:latest
    container_name: technitium-dns
    hostname: dns.[YOUR.DOMAIN]
    ports:
      - "5380:5380/tcp"
      - "53443:53443/tcp"
      - "53:53/udp"
      - "53:53/tcp"
      - "853:853/tcp" 
      - "853:853/udp" 
      - "443:443/tcp" 
      - "443:443/udp" 
      - "80:80/tcp"   
    volumes:
      - ./config:/etc/dns   # Use a local folder for config

    environment:
      - DNS_SERVER_DOMAIN=dns.[YOUR.DOMAIN]
    restart: unless-stopped
```

We will be giving our DNS server a certificate for use with https (skip this if you don't want to use HTTPS)

NOTE: We won't actually implement the certificate until after we gain access to the GUI.   

### Self-signed method(simplest):

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ~/cert/private.key -out ~/cert/certificate.crt
```

```
openssl pkcs12 -export -out ~/network-services/dns/config/ssl/certificate.pfx -inkey ~/cert/private.key -in ~/cert/certificate.crt
```

and you're done!

### Cloudflare Origin Certificate Method:

This is the most useful method if you are only concerned with your certificate being valid and trusted by Cloudflare's servers. You must get an origin certificate and private key as a pair to create the required .pfx file for Technitium. This is available at cloudflare under 'SSL/TLS settings > origin certificate', click create certificate, use defaults except make sure you specify the correct domain in the creation process. this will give you a certificate and private key. save the certificate as a .pem file and the private key as a .key file on your computer as a backup.

once you have the certificate.pem and private.key, put them into a directory on your DNS server's machine and generate the .pfx token:
```
mkdir ~/cert
nano ~/cert/private.key # paste the private key here
nano ~/cert/certificate.pem # paste the origin certificate here
sudo apt install openssl
mkdir ~/network-services/dns/config/
mkdir ~/network-services/dns/config/ssl
sudo openssl pkcs12 -export -out ~/network-services/dns/config/ssl/certificate.pfx -inkey ~/cert/private.key -in ~/cert/certificate.pem
```

### Trusted Certificate method:
   
   1. Obtain an API token from Cloudflare
      My Profile > API Tokens > Create Token, 
      choose "Edit zone DNS" template, 
      specify domain under "Zone resources", 
      click continue and then create token.

   2. Save the token to a file named cloudflare.ini in a secure place as backup.
      
      Now let's paste it onto your Ubuntu machine: 

```
mkdir ~/cert
nano ~/cert/cloudflare.ini # (paste the token here as a key/value pair with this key: 'dns_cloudflare_api_token = ' prepending the token. format: dns_cloudflare_api_token = [your api token]) 
```

```
chmod 600 ~/cert/cloudflare.ini # (required by certbot to avoid security flag)
```

   3. Install Certbot and it's Cloudflare plugin
```
sudo apt update
sudo apt install certbot python3-certbot-dns-cloudflare
```
   4. Replace placeholders and request the certificate:
```
sudo certbot certonly \
 --dns-cloudflare \
 --dns-cloudflare-credentials ~/cert/cloudflare.ini \
 -d dns.[YOUR.DOMAIN] \
 --agree-tos \
 -m [YOUR EMAIL ADDRESS]
```
   5. Convert the Certbot output to a .pfx for Technitium's requirement using OpenSSL (it will prompt you for a password,          make note of your choice for later):
```
mkdir ~/network-services/dns/config/
mkdir ~/network-services/dns/config/ssl/
sudo openssl pkcs12 -export -out ~/network-services/dns/config/ssl/certificate.pfx \
 -inkey /etc/letsencrypt/live/dns.[YOUR.DOMAIN]/privkey.pem \
 -in /etc/letsencrypt/live/dns.[YOUR.DOMAIN]/fullchain.pem
```
You may now start the service:
```
cd ~/network-services/dns/
sudo docker-compose up -d
```

## DDNS 
 
1. For the Cloudflare DDNS server we must get an API token, as previously stated here are the steps:
```
      My Profile > API Tokens > Create Token
      choose "Edit zone DNS" template
      specify domain under "Zone resources"
      click continue and then create token.
```


2. Prepare the DDNS server YAML

```
nano ~/network-services/ddns/docker-compose.yml
```

   Paste this into the .YAML and replace the api_key, zone, and domain with your info:

```
version: "3.7"

services:
  cloudflare-ddns-root:
    image: oznu/cloudflare-ddns:latest
    container_name: cloudflare-ddns-root
    restart: unless-stopped
    environment:
      - API_KEY=[YOUR API KEY]
      - ZONE=[YOUR.DOMAIN]
      - DOMAIN=[YOUR.DOMAIN]
      - PROXIED=false

 ~ if you would like to add a subdomain to your public A records you may do so by adding this into the .YML file, under the one we pasted previously, 
   just replace [SUBDOMAIN] with something (for example: vpn for a vpn service):
 
  cloudflare-ddns-dns:
    image: oznu/cloudflare-ddns:latest
    container_name: cloudflare-ddns-dns
    restart: unless-stopped
    environment:
      - API_KEY=[YOUR API KEY]
      - ZONE=[YOUR.DOMAIN]
      - SUBDOMAIN=[SUBDOMAIN]
      - PROXIED=false
```

 ~ you can now run sudo docker-compose up -d within the ddns directory to have your IP address dynamically update within your domain's A records
 
- Configuring Technitium:

 ~ Now let's navigate to our DNS server's GUI and change some settings:

  1. Go to your browser and in the URL input box type: [YOUR DNS SERVER's IP]:5380

  2. Create a password

  3. navigate to settings > general and uncheck/disable DNSSEC to avoid any issues

  4. Enable HTTPS (optional): 
     navigate to settings > web service: 
     check the enable https box
     change the HTTPS Port to 443 for simplicity or leave it as is, you can also give a totally different unused one as well just make sure you specify
     the port in your YAML file as the others are.
     you can also check the box to redirect http to https, as recommended for security.
     specify your certificate name and location in the TLS Certificate Input(if you followed the steps assiduously, input this: ./ssl/certificate.pfx)
     in the next input box "TLS Certificate Password" give the export password you created with your .pfx certificate.
     you can now access your site using https.  

  5. Specify forwarders:
     under settings > Proxy & Forwarders insert some public DNS server IPs to use for forwarding. This is so a device using your DNS server will still be able to
     reach a URL not listed in your local DNS server's records. 

  6. Change your home router's DNS server to your local one, or just change your devices' DNS server if you prefer. This step is crucial for accessing your custom 
     DNS records.
    
 ~ Now let's add some records:

   1. Click on the zones tab.
   
   2. Click add zone in the upper right corner.

   3. input your domain name (example.com) and save.
   
   4. now in the lists of zones you will see your domain, you can click the name to open the zone's tab where we'll add our subdomains(if it didn't redirect
      automatically).

   5. in the upper region you will see add record, click it and make sure A record is selected and then we'll start with our first record for DNS. Within the name
      field insert 'dns' and specify the IP address of your DNS server in the respective field. Save this record and now you will be able to access the technetium
      GUI or even SSH to the server using the subdomain URL. Repeat these steps for any IP you want to assign a subdomain to, including public IPs. For example you
      could assign a domain to cloudflare's DNS server (1.1.1.1 > clouddns.your.domain), or maybe you want a subdomain for your vm!

~ You now have a complete home lab DNS set up! It's worth noting that you can obtain block lists(for ads, scams, etc.) to give your technitium server, much like you
  would a pihole machine. find them here: https://github.com/hagezi/dns-blocklists, choose the level of blocking you want and make sure you use the list designated for 
  technitium. In Techinitium's GUI, navigate to the 'Blocked' tab and select import. copy and paste the list of URLs here.

   
