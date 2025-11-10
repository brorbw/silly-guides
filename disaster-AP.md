# Guide for setting up an OrangePI emergency router with AP
If you find yourself in the situation where you have no internet and need to bridge your network with an internet connection on your iphone this guide is for you.
This guide in particular is for an OrangePI zero 3, but you can use any computer. If you don't want an AP you can make small changes to the configuration, specifically ignore the hostap configuration and the bridge device. You would also need to change the `nftables` rules to match on `iff <ethernet_interface>` rather than the bride.


# Prerequisites

- Armbian (debian flavour) flashed to the Pi
- Orange Pi zero 3 (or similar)

# Connecting to the Pi
Initially you will need to connect to the Pi. If you have a DCHP server in your network this is easy, plug it in and connect to the IP. If you don't have a DCHP server jump to the TBD section which describes the steps needed to create a DCHP server locally that you can use to connect to the Pi. These steps also includes a guide for sharing your local tethered connection to download dependencies and packages onto the Pi before continuing with the setup.

If you use the Armbian image the default user is `root` and the pass is `1234`. You will be prompted to make a new one.

# Installing dependencies
First update and upgrade using apt
```sh
sudo apt update && sudo apt upgrade -y
```
Then install the dependencies
```sh
sudo apt install hostapd nftables iw wireless-tools -y

# Possible extra dependencies if you are not using armbian or if they are missing
sudo apt install linux-modules-extra-$(uname -r) usbmuxd libimobiledevice6 libimobiledevice-utils ipheth-utils -y
```

# Configuration

## Interfaces:
By default all iphone devices will be called `enx*` as a prefix.

If you don't want an AP you should also omit the `20-br0.netdev` and the `21-end0.network`, rename the `21-br0.network` to `20-end0.network` and also change `Name=br0` to `Name=end0`.

#### The iPhone network(`/etc/systemd/network/10-enx.network`):
```conf
[Match]
Name=enx*

[Network]
DHCP=yes
DNS=1.1.1.1 8.8.8.8
RouteMetric=10
```

#### The bridge interface(`/etc/systemd/network/20-br0.netdev`):
```conf
# /etc/systemd/network/20-br0.netdev
[NetDev]
Name=br0
Kind=bridge
```

#### The bridge network(`/etc/systemd/network/21-br0.network`):
```conf
[Match]
Name=br0

[Network]
Address=192.168.50.1/24
IPForward=yes
DHCPServer=yes

[DHCPServer]
PoolOffset=10
PoolSize=190
DNS=1.1.1.1 8.8.8.8
DefaultLeaseTimeSec=43200
MaxLeaseTimeSec=86400
```

#### The lan network(`/etc/systemd/network/21-end0.network`):
On the OrangePi Zero 3, the ethernet interface is `end0`. If you use this guide for another device, you should use the matching interface.
```conf
[Match]
Name=end0
[Network]
Bridge=br0
```

```sh
sudo systemctl enable --now systemd-networkd systemd-resolved
# In case NetworkManager is running disable it now
sudo systemctl disable --now NetworkManager || true
```

## AP
This step can be skipped if you don't want to enable the AP. This could be for low power consumption. 

#### Configuration 
Configuration file (`/etc/hostapd/hostpad.conf`)
```conf
interface=wlan0
bridge=br0
driver=nl80211
ssid=<REPLACE WITH AP NAME>
country_code=<REPLACE WITH YOUR CONTRY CODE>
hw_mode=g
channel=6
ieee80211n=1
wmm_enabled=1
auth_algs=1
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_passphrase=<REPLACE WITH PASSWORD?
rsn_pairwise=CCMP
```

Now add the config file so `hostapd` can find it.
```
sudo sed -i 's|^#\?DAEMON_CONF=.*|DAEMON_CONF="/etc/hostapd/hostapd.conf"|' /etc/default/hostapd
```
If you want to edit manually add the line:
```conf
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```
To the file `/etc/default/hostapd`.

### Starting the AP
```sh
sudo systemctl enable --now hostapd
```

## nftables
The default build of `system-networkd` does not support `IPMasqurade` as it is compiled without iptables and nftables so we need to create a rule to manually forward the connection to the iPhone. I use `iffname` and `oifname` over `iff` and `oif` to avoid interface not found errors. Albeit being slower, the `*name` keywords do not depend on the interface existing when the rules are created, which happens because the rules are applied before the `br0` interface is created. `iff` and `oif` can be used if the AP is omitted, hence no bridge.

#### Configuration
Configure the file `/etc/nftables.conf`
```conf
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iifname "lo" accept

        # DHCP from LAN clients
        iifname "br0" udp dport {67,68} accept

        # SSH from LAN (optional)
        iifname "br0" tcp dport 22 accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;

        # LAN -> WAN
        iifname "br0" oifname "enx*" accept

        # Return traffic
        iifname "enx*" oifname "br0" ct state established,related accept

        # To isolate LAN clients change accept to drop
        iifname "br0" oifname "br0" accept
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 192.168.50.0/24 oifname "enx*" masquerade
    }
}
```
Or if you skipped the AP setup and omitted the bridge:
```conf
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iifname "lo" accept

        # DHCP from LAN clients
        iif "end0" udp dport {67,68} accept

        # SSH from LAN (optional)
        iif "end0" tcp dport 22 accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;

        # LAN -> WAN
        iif "end0" oifname "enx*" accept

        # Return traffic
        iifname "enx*" oif "end0" ct state established,related accept

        # To isolate LAN clients change accept to drop
        iif "end0" oif "end0" accept 
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 192.168.50.0/24 oifname "enx*" masquerade
    }
}
```


```sh
sudo systemctl enable --now nftables
```

Now you should have networking on the ethernet port and optionally on the AP.

# AP

If you want to temporarily disable or enable the AP use the following steps.

#### Disable

```sh
sudo systemctl disable --now hostapd
sudo ip link set wlan0 down
```

#### Enable

```sh
sudo ip link set wlan0 up
sudo systemctl enable --now hostapd
```

# Sharing network connection to install deps
TBD
