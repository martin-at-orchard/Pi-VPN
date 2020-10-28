# Orchard Recovery Center Raspberry Pi VPN

## Description

This project uses a Raspberry Pi 3B and a 32 GByte SD card. It is connected as in a headless configuration with only a wired Ethernet connection. WireGuard is used to provide the VPN services.

## Raspian Installation

Install Raspian Buster Lite on the SD Card.

1. Download the latest version of Raspbian Buster Lite from [Raspberry Pi Downloads](https://www.raspberrypi.org/downloads/raspbian/).
1. Format the SD Card.
   1. Windows or Mac - Use SD Card Formatter from [SD Association](https://www.sdcard.org/downloads/formatter/).
1. Burn the Raspbian Buster Lite image to the SD Card.
   1. Windows or Mac - Use balaenaEtcher from [balena](https://www.balena.io/etcher/).
   1. There is **NO** need to unzip the Raspbian Buster Lite file as Etcher will handle it.
1. Create an empty file in the root partition of the SD Card with the name **ssh**.
1. Put the SD Card in the Raspberry Pi.
1. Connect up a wired Internet connection.
1. Power up the Raspberry Pi.
1. Find the IP Address for the Raspberry Pi. Will be the IP Address with port 22 (ssh) open.
   1. Windows - Use [Angry IP Scanner](https://angryip.org/).
   1. Mac - Use Terminal.
      1. In Finder open Terminal which is found in the Applications -> Utilities Folder.
      1. Enter the command `arp -a`.
         1. Raspberry Pi's will have MAC addresses with the format **B8:27:EB:xx:xx:xx** or **DC:A6:32:xx:xx:xx**.
1. SSH into the Raspberry Pi by connecting to the IP Address that was found above.
   1. Windows - Use [PuTTY](https://www.putty.org/).
   1. Mac - Use Terminal.
1. Use Userid: **pi** and Password: **raspberry**.
   ```script
   login as: pi
   pi@IP-ADDRESS's password: raspberry
   ```
1. Mount the /tmp, /var/tmp, /var/log directories into RAM (optional, but will reduce the ware on the micro SD card, but the log files will be lost on reboot and the available RAM will be reduced by 230 MB).
   ```shell
   sudo nano /etc/fstab
   ```
   1. Once editing the file enter the following at the bottom of the file.
   ```shell
   /tmpfs /tmp     tmpfs defaults,noatime,nosuid,size=100m           0 0
   /tmpfs /var/tmp tmpfs defaults,noatime,nosuid,size=30m            0 0
   /tmpfs /var/log tmpfs defaults,noatime,nosuid,size=100m,mode=0755 0 0
   ```
   1. Press **^o** then **Enter** then **^x** to save and exit.

1. Configure the Raspberry Pi
   ```
   $ sudo raspi-config
   ```
   1. Change User Password - Change password for the 'pi' user
   1. Network Options - Configure network settings
      1. Hostname - Set the visible name for this Pi on a network
      1. Wireless LAN - Enter SSID and passphrase (**OPTIONAL**)
   1. Localization Options - Set up language and regional settings to match your location
      1. Change Locale - Set up language and regional settings to match your location
      1. Change Time Zone - Set up timezone to match your location
      1. Change Keyboard Layout - Set the keyboard layout to match your keyboard
   1. Finish to leave the configuration, but **DON'T** reboot just yet.
1. Edit the pi user .bashrc file.
   ```shell
   nano ~/.bashrc
   ```
   1. Enter the above aliases to avoid making mistakes.
      ```shell
      alias rm='rm -i'
      alias cp='cp -i'
      alias mv='mv -i'
      ```
1. **Note** Optionally set a Static IP address
   ```script
   sudo nano /etc/dhcpcd.conf
   ```
   1. Scroll down to the commented out section Example static IP configuration and modify it accordingly
    
1. Update the Raspberry Pi Software.
   ```
   $ sudo apt update
   $ sudo apt upgrade -y
   ```
1. Reboot the pi.
```
$ sudo shutdown -r now
```

## WireGuard Installation

This setup is based on a blog post from [The Engineer's Workshop](https://engineerworkshop.com/2020/02/20/how-to-set-up-wireguard-on-a-raspberry-pi/) with modifications for used at Orchard Recovery Center

1. Add the Debian distro to the Raspberry Pi apt source.
```
$ echo "deb http://deb.debian.org/debian/ unstable main" | sudo tee --append /etc/apt/sources.list
```
1. Install the Debian distro keys.
```
$ sudo apt-key adv --keyserver   keyserver.ubuntu.com --recv-keys 04EE7237B7D453EC
$ sudo apt-key adv --keyserver   keyserver.ubuntu.com --recv-keys 648ACFD622F3D138
```
1. Prevent Pi from using the Debian distro for normal Raspbian packages.
```
$ sudo sh -c 'printf "Package: *\nPin: release a=unstable\nPin-Priority: 90\n" > /etc/apt/preferences.d/limit-unstable'
```
1. Update the package list.
```
$ sudo apt-get update
```
1. Install WireGuard.
```
$ sudo apt install wireguard
```
1. Wait for the installation to finish (could take some time).
1. Generate security keys to ensure that only valid users can connect (the client keys need to be generated for as many clients as needed). **NOTE From this point on you will be root**.
```
$ sudo su
# cd /etc/wireguard
# umask 077
# wg genkey | tee server_private_key | wg pubkey > server_public_key
# wg genkey | tee client_private_key_1 | wg pubkey > client_public_key_1
# wg genkey | tee client_private_key_2 | wg pubkey > client_public_key_2
```
1. Retrieve the keys generated above as they will be needed during editing the configuration file.
```
$ cat filename
```
1. Create the configuration file.
```
$ nano wg0.conf
```
1. Add the following configuration changing the IP addresses and the keys as required.
```
[Interface]
Address = 10.253.3.1/24
SaveConfig = true
PrivateKey = <insert server_private_key>
ListenPort = 51900

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <insert client_public_key_1>
AllowedIPs = 10.253.3.2/32

[Peer]
PublicKey = <insert client_public_key_2>
AllowedIPs = 10.253.3.3/32
```
1. Enable IP Forwarding on the Server by uncommenting the line **net.ipv4.ip_forward=1** in the sysctl.conf file
```
$ nano /etc/sysctl.conf
```
1. Start up WireGuard
```
$ systemctl enable wg-quick@wg0
$ chown -R root:root /etc/wireguard/
$ chmod -R og-rwx /etc/wireguard/*
```
1. Reboot the Raspberry pi
```
$ shutdown -r now
```

## Port forward the router to the Raspberry Pi

The Raspberry Pi will listen on port 51900 (default), this is the internal port. The external port (Public IP Address) can be available port.  This will be different for all routers. See your router configuration.
```
EXTERNAL-IP-ADDRESS EXTERNAL-PORT forwarded to RASPBERRY-PI-IP-ADDRESS 51900
```

## Client configuration

On each of the clients create a configuration file **wg0-client.conf**. Make sure to change the IP address and ports as required.
```
[Interface]
Address = 10.253.3.2/32
PrivateKey = <insert client_private_key>
DNS = 1.1.1.1

[Peer]
PublicKey = <insert server_public_key>
Endpoint = <insert vpn_server_address>:51900
AllowedIPs = 0.0.0.0/0, ::/0
```

### Install Client software

1. Go to [WireGuard](https://www.wireguard.com/install/) and get the appropriate installation file.
1. Install WireGuard.
1. Add the above configuration file to the WireGuard client with the **Add Tunnel** button.
1. Click the **Activate** button to enable the VPN.

### Access Orchard Documents

1. Windows - Connect to \\192.168.0.39\OrchardDocuments
1. Mac - Connect to Server \\192.168.0.39\OrchardDocuments
