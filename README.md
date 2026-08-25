# Building a Custom Unbound for Firewalla

Firewalla routers ship with a very old version of Unbound (**1.13.0** on the newest FWG+ image). The current upstream release is **1.26.0**, which includes significantly more hardening features.

This guide walks through pulling the Unbound source, building it directly on the Firewalla, and running it as a separate service alongside (or in place of) the stock Unbound.

> **Use at your own risk.** Tested on a Firewalla Gold Plus, but should work on Orange and all Gold models. I wouldn't try this on a Purple or older but YMMV.

## Overview

To update Unbound on Firewalla, you need to:

1. Pull the source code
2. Build it on the Firewalla itself
3. Run it as a separate service
4. Stop and disable the stock Firewalla Unbound

## 1. Create a Working Directory

SSH into the Firewalla, then create a directory for the custom build:

```bash
mkdir -p /home/pi/unbound-custom
cd /home/pi/unbound-custom
```

## 2. Install Build Dependencies

```bash
sudo /home/pi/firewalla/scripts/apt-get.sh update
sudo /home/pi/firewalla/scripts/apt-get.sh install build-essential libssl-dev libevent-dev libexpat1-dev
```

## 3. Download and Build Unbound

> Note: update the version number below if you're not building 1.26.0.

```bash
wget https://www.nlnetlabs.nl/downloads/unbound/unbound-latest.tar.gz
tar -xzf unbound-latest.tar.gz
cd unbound-1.26.0
./configure --prefix=/home/pi/unbound-custom --with-libevent --with-ssl
make
make install
```

Once complete, the custom Unbound binary is available at:

```
/home/pi/unbound-custom/sbin
```

## 4. Configure `unbound.conf`

You'll need an `unbound.conf` in `/home/pi/unbound-custom`. The simplest approach is to copy over the stock config as a starting point:

```bash
cp /home/pi/.firewalla/run/unbound/unbound.conf /home/pi/unbound-custom/
```

Then update the paths inside it to point to the new custom install location (see below).

### Generate a new root key and root hints

```bash
/home/pi/unbound-custom/sbin/unbound-anchor -a "/home/pi/unbound-custom/etc/unbound/root.key"
sudo wget https://www.internic.net/domain/named.root -O /home/pi/unbound-custom/etc/unbound/root.hints
```

### Update directory paths in `unbound.conf`

```
directory: "/home/pi/unbound-custom/etc/unbound"
pidfile: "/home/pi/unbound-custom/etc/unbound/unbound.pid"
```

Make sure any other paths referenced in the config (root key, root hints, etc.) are also updated to point at `/home/pi/unbound-custom/...` rather than the stock locations.

## 5. Create a systemd Service

Create the service file:

```bash
sudo nano /etc/systemd/system/unbound-custom.service
```

Contents:

```ini
[Unit]
Description=Custom Unbound 1.26.0 DNS Server
After=network.target

[Service]
Type=simple
User=pi
ExecStart=/home/pi/unbound-custom/sbin/unbound -d -c /home/pi/unbound-custom/unbound.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## 6. Switch Over to the Custom Build

Stop and disable the stock Unbound:

```bash
sudo systemctl stop unbound
sudo systemctl disable unbound
```

Start and enable the custom Unbound:

```bash
sudo systemctl start unbound-custom
sudo systemctl enable unbound-custom
```

### Reverting to Stock

To switch back, just reverse the steps:

```bash
sudo systemctl stop unbound-custom
sudo systemctl disable unbound-custom
sudo systemctl start unbound
sudo systemctl enable unbound
```

If you are interested in this you may be interested in my other Unbound/Firewalla repos.  
- [Configuring Unbound on Firewalla to use DNS-over-TLS](https://github.com/upmcplanetracker/firewalla-unbound-DoT-config)
- [Adding Huge Blocklists to FW via Unbound](https://github.com/upmcplanetracker/firewalla-huge-blocklists)
- [Replacing FW Timekeeper with NTS via ntpd-rs](https://github.com/upmcplanetracker/ntpd-rs-nts-for-firewalla)
