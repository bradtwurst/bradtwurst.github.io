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
