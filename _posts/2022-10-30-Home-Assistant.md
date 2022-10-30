---
layout: post
title: Home Assistant
categories: ["iot", "debian", "home assistant", "coral.ai", "cockpit", "certificates", "podman"]
---

## Main page describing my Home Assistant setup

...So I have a good place to review what I didn't do right...

## The current configuration
* Dell Optiplex 7040 - total - $251
  - renewed - $150
    - Intel Quad Core i5-6500T - 3.1GHz 
    - 8GB DDR3 SRAM
    - 256 GB SSD (replaced with a larger HDD)
  - Leven JM300 M.2 SSD 480GB - Sata 3 - $28
  - WD Blue 1TB HDD - 5400 RPM Sata3 - $40
  - Coral.ai M.2 TPU Accelerator - $33
  - Debian GNU/Linux 11 (bullseye)
    - a lot of issue getting debian to boot after installation
    - needed to do 'expert install' 
    - saying 'yes' when GRUB install asks about 'buggy' uefi firmware
    - see   {% post_url 2022-10-30-debian-optiplex-7030 %}
    - the next steps were
      - install coral.ai drivers -- see [below](#coral-ai-driver-install)
      - remote web admin -- see [below](#cockpit-install)
      - get rid of untrusted certs warnings in browser -- see [below](#manage-untrusted-cert)
      - container manager -- see [below](#install-podman)
      - home assistant container install -- see [below](#install-home-assistant)


## Coral AI Driver Install ##

I following the guide located [here](https://coral.ai/docs/m2/get-started){:target="_blank"}{:rel="noopener noreferrer"}

The main steps in section _2a: On Linux_
- Confirm linux version 

```shell
uname -r
# my install returned 5.10.0-19-amd64
```

- if 4.19 or higher, check if the pre-built Apex driver is installed

```shell
lsmod | grep apex
# my install returned no listing
```

- if the apex driver is listed, it will need to be addressed / removed
- I didn't have this issue - so I don't have knowledge
- Add the coral.ai packages

```shell
# I needed to install curl as the minimal debian install didn't have it
#   - otherwise, the repo won't be properly signed and supported
sudo apt-get install curl

# add the coral.ai debian package repository 
echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | sudo tee /etc/apt/sources.list.d/coral-edgetpu.list

# add the repository key
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# normal 'update the repo packages'
sudo apt-get update

# install the TPU runtime packages
sudo apt-get install gasket-dkms libedgetpu1-std

# add permissions for the apex group
sudo sh -c "echo 'SUBSYSTEM==\"apex\", MODE=\"0660\", GROUP=\"apex\"' >> /etc/udev/rules.d/65-apex.rules"

sudo groupadd apex

sudo adduser $USER apex
```

- reboot the system
- verify the module is detected and the pcie driver is loaded

```shell
lspci -nn | grep 089a
# my install returned a device with the 089a id
# 02:00.0 System peripheral [0880]: Global Unichip Corp. Coral Edge TPU [1ac1:089a]

ls /dev/apex_0
# my installed returned
# /dev/apex_0

```


## Cockpit Install ##

Per Cockpit's website...

Cockpit is a web-based graphical interface for servers, intended for everyone, especially those who are:
- new to Linux (including Windows admins)
- familiar with Linux (and want an easy, graphical way to administer servers)
- expert admins (who mainly use other tools but want an overview on individual systems)

following the instructions on [Installing Cockpit on Debian 11](https://www.howtoforge.com/how-to-install-cockpit-on-debian-11/){:target="_blank"}{:rel="noopener noreferrer"}

```shell

# by default cockpit is included in Debian 11 default package repository
apt-get install cockpit -y

# after installing cockpit, we'll also install the podman plugin (for use later)
apt-get install cockpit-podman -y

# after successful install, start the service and enable it to auto-start on system reboot
systemctl start cockpit
systemctl enable cockpit

# you can check the status with
systemctl status cockpit

# i don't have UFW firewall up so no changes to ufw
```

the web interface should now be available at *http://your-server-ip:9090*

## Manage Untrusted Cert ##

One of the annoying things is to deal with the 'this web certificate is not trusted'.

So let's 'fix' that.

Since I am running this server on my private network with a non-public top level domain (TLD), I can't use a normal certificate authority, as they require a public TLD.

Instead, we'll set up our own private CA by following the instructions [here](https://dgu2000.medium.com/working-with-self-signed-certificates-in-chrome-walkthrough-edition-a238486e6858){:target="_blank"}{:rel="noopener noreferrer"}

* create the CA

```shell
# Generate an RSA private key of size 2048:

openssl genrsa -des3 -out rootCA.key 2048

# Generate a root certificate valid for two years:

openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 730 -out rootCA.pem

#To check just created root certificate:

openssl x509 -in rootCA.pem -text -noout
```

* create the certificate signing request

```shell
# First, create a private key to be used during the certificate signing process:

openssl genrsa -out tls.key 2048

# Use the private key to create a certificate signing request:

openssl req -new -key tls.key -out tls.csr
```

* create a config file openssl.cnf
   * Edit the domain(s) listed under the *alt_names* section, be sure they match the domain name you want to use.

```ini
# Extensions to add to a certificate request
basicConstraints       = CA:FALSE
authorityKeyIdentifier = keyid:always, issuer:always
keyUsage               = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
subjectAltName         = @alt_names
[ alt_names ]
DNS.1 = *.yourdomain.home <== use your non-public TLD here
```

* sign the certificate request using the CA

```shell

# sign the CSR
openssl x509 -req \
    -in tls.csr \
    -CA rootCA.pem \
    -CAkey rootCA.key \
    -CAcreateserial \
    -out tls.crt \
    -days 730 \
    -sha256 \
    -extfile openssl.cnf

# verify the cert
openssl verify -CAfile rootCA.pem -verify_hostname somehost.yourdomain.home tls.crt
```

* Add the CA to the trusted CA's
   - copy the PEM from your server to your Windows PC.  this can be done manually by copying the content within the .pem file on the server and pasting it into a new file using a windows text editor
   - open an administrator command prompt
   - execute the following command

```shell
certutil --addstore -f "ROOT" <path to .pem file>
```

* Add the cert to the cockpit service

```shell

# join the tls.crt and tls.key into a single file
cat tls.crt tls.key > cockpit.crt

# copy it to the cockpit config area
sudo cp cockpit.crt /etc/cockpit/ws-certs.d

# restart the service
sudo systemctl restart cockpit

# validate that the cert is being used
sudo remotectl certificate

# should list the cockpit.crt
```


## Install Podman ##

Per [podman website](https://podman.io){:target="_blank"}{:rel="noopener noreferrer"},

Podman is a daemonless container engine for developing, managing, and running OCI Containers on your Linux System. Containers can either be run as root or in rootless mode. Simply put: alias docker=podman

To install,

```shell
 
sudo apt-get -y install podman

```


## Install Home Assistant ##

This is done as a container.

Following the docs [here](https://www.redhat.com/sysadmin/automate-your-home){:target="_blank"}{:rel="noopener noreferrer"}

```shell
# create some of the folders that will be used for persistant info

# i mounted the HDD (spinning rust) at /data
# hass_config is for the home assistant config data
# hass_media is for the home assistant media data

# setup the container

# I changed a few things from the linked docs

# # --network=host wasn't allowing external (non-localhost) connections
#    using -p 8123:8123 instead
# # there was an error regarding 'dhcp discovery - operation not permitted'
#    suggestion is that the --cap-add... parameter should be used
# # --pull=always to get latest home assistant on startup
# # added read-only volume for /etc/localtime

podman run -d --name homeassistant --cap-add=CAP_NET_RAW,CAP_NET_BIND_SERVICE --restart=unless-stopped -p 8123:8123 -v /etc/localtime:/etc/localtime:ro -v ~/data/container/hass_config:/config:Z -v ~/data/container/hass_media:/media:Z --pull=always homeassistant/home-assistant:stable

```

* then connect to *http://<server ip>:8123*

* you'll be asked to setup the following
  - setup a user account
    - name, username, password
  - home assistant configuration
    - HA name, location, time zone, altitude, and currency
    - [elevation finder](https://www.freemaptools.com/elevation-finder.htm){:target="_blank"}{:rel="noopener noreferrer"} can be used to find elevation
  - request to share analytics
  - devices and services to add
    - that can be done later
    

