# XCarlink
Music server in a car

# XCarlink

## Install Raspberry Pi OS Lite

- [Install Raspberry Pi OS using Raspberry Pi Imager](https://www.raspberrypi.org/software) (ISO is downloaded with the app)
- [Raspberry Pi Documentation](https://www.raspberrypi.org/documentation)
- [Setting up a Raspberry Pi headless](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md)
- [SSH (Secure Shell)](https://www.raspberrypi.org/documentation/remote-access/ssh/README.md)
  - Copy `./boot/ssh` file to the SD Card `/boot` folder
  - `alias rpi='ssh pi@raspberrypi.local'`
  - `alias xcarlink='ssh pi@xcarlink.local'`
  - Connection (default password: `raspberry`)
  - [SSH USING iOS](https://www.raspberrypi.org/documentation/remote-access/ssh/ios.md)

---

## Config

Custom aliases
```sh
(host) scp ./home/pi/.bash_aliases pi@raspberrypi.local:~/.bash_aliases
```

[Updating and upgrading Raspberry Pi OS](https://www.raspberrypi.org/documentation/raspbian/updating.md)
  ```sh
  sudo apt update
  sudo apt full-upgrade
  sudo apt install vim
  ```
[raspi-config](https://www.raspberrypi.org/documentation/configuration/raspi-config.md)
- `sudo raspi-config`

1. System Options
   - S2 Audio: 3.5mm jack
   - S3 Password: ChangeMe
   - S4 Hostname: xcarlink
5. Localisation Options
   - L2 Timezone: Europe/Paris
   - L4 WLAN Country: FR
6. Advanced Options
   - A1 Expand Filesystem
8. Update
   - `sudo apt autoremove`

```sh
sudo reboot
```

---

## Test a MP3 sound sample

1. Install package to read mp3
```sh
sudo apt update && sudo apt install mpg123
```
2. Copy sound sample
```sh
mkdir -p ~/music
(host) scp ./srv/music/*.mp3 pi@xcarlink.local:~/music
sudo mv ~/music /srv/music
sudo chown -R root:root /srv/music
```
3. Test sound
```sh
mpg123 /srv/music/sample.mp3
```

---

## OwnTone

- [Github OwnTone](https://github.com/owntone/owntone-server)
  - [Installation instructions for OwnTone](https://github.com/owntone/owntone-server/blob/master/INSTALL.md)
  - [owntone server (iTunes server)](https://www.raspberrypi.org/forums/viewtopic.php?t=49928)

### How to install

1. Add repository key
```sh
wget -q -O - http://www.gyfgafguf.dk/raspbian/owntone.gpg | sudo apt-key add -
```
2. Add this line to `/etc/apt/sources.list`
```sh
sudo cp /etc/apt/sources.list /etc/apt/sources.list.org
echo "deb http://www.gyfgafguf.dk/raspbian/owntone/ buster contrib" | sudo tee -a /etc/apt/sources.list
```
3. Run
```sh
sudo apt update && sudo apt install owntone
```

### Getting started

1. Edit the configuration file to suit your needs
```sh
sudo cp /etc/owntone.conf /etc/owntone.conf.org
sudo vim /etc/owntone.conf
L32         admin_password = ""
L76         name = "My Music on XCarlink"
L227        nickname = "XCarlink"
L230        type = "pulseaudio"
L235        server = "localhost"
```
2. Start or restart the server
```sh
sudo service owntone restart
sudo service owntone status
```
3. Wait for the library scan to complete. You can follow the progress with
```sh
tail -f /var/log/owntone.log
```

If you are going to use a remote app, pair it by going to http://owntone.local:3689 and find the settings

## OwnTone and Pulseaudio

[System Mode with Bluetooth support](https://github.com/owntone/owntone-server/blob/master/README_PULSE.md#system-mode-with-bluetooth-support)

### Setting up Pulseaudio

1. Install packages
```sh
sudo apt update && sudo apt install pulseaudio pulseaudio-module-bluetooth ofono
```
2. Remove client execution
```sh
sudo cp /etc/pulse/client.conf /etc/pulse/client.conf.org
sudo vim /etc/pulse/client.conf
L25 autospawn = no
mkdir -p $HOME/.config/pulse
cp /etc/pulse/client.conf $HOME/.config/pulse/client.conf
```
3. Start Pulseaudio on boot
```sh
sudo vim /etc/systemd/system/pulseaudio.service
Copy/Paste ./etc/systemd/system/pulseaudio.service
```
4. Bluetooth support, add lines
```sh
sudo cp /etc/pulse/system.pa /etc/pulse/system.pa.org
sudo vim  /etc/pulse/system.pa
#### Enable Bluetooth
.ifexists module-bluetooth-discover.so
load-module module-bluetooth-discover autodetect_mtu=yes
.endif
```
5. Make Pulseaudio communicates with the Bluetooth daemon through D-Bus
```sh
sudo adduser pulse audio
sudo adduser pulse bluetooth
```
6. Edit and change the policy for `<policy context="default">` from `deny` to `allow`
```sh
sudo cp /etc/dbus-1/system.d/bluetooth.conf /etc/dbus-1/system.d/bluetooth.conf.org
sudo vim /etc/dbus-1/system.d/bluetooth.conf
L40 <allow send_destination="org.bluez"/>
Add lines:
  <policy context="default">
    <allow send_destination="org.ofono"/>
  </policy>
```
7. Restart the bus
```sh
sudo service dbus restart
sudo service dbus status
```
8. Enable system mode on boot
```sh
sudo systemctl --user stop pulseaudio.service pulseaudio.socket
sudo systemctl --global disable pulseaudio.service pulseaudio.socket
sudo systemctl enable bluetooth
sudo systemctl enable ofono
sudo systemctl enable pulseaudio
sudo reboot
```
9. Check that the Bluetooth module is loaded
```sh
pactl list modules short | grep -i bluetooth
```

### Setting up the server

1. Add the user the server is running
```sh
sudo adduser root pulse-access
sudo adduser pulse pulse-access
```
2. Restart and check the logs
```sh
sudo service pulseaudio restart
sudo service pulseaudio status
journalctl -u pulseaudio.service
```

### Adding a Bluetooth device

To connect with the device, run `bluetoothctl` and then:
```sh
power on
agent on
scan on
**Note MAC address of BT Speaker**
pair [MAC address]
**Type Pin if prompted**
trust [MAC address]
connect [MAC address]
```

Now the speaker should appear. You can also verify that Pulseaudio has detected the speaker with
```sh
pactl list sinks short
```

---

## [Setting up a Raspberry Pi as a routed wireless access point](https://www.raspberrypi.org/documentation/configuration/wireless/access-point-routed.md)

1. Install the access point and network management software
```sh
sudo apt update && sudo apt install hostapd
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo apt update && sudo apt install dnsmasq
sudo DEBIAN_FRONTEND=noninteractive apt install -y netfilter-persistent iptables-persistent
```
2. Set up the network router
```sh
sudo cp /etc/dhcpcd.conf /etc/dhcpcd.conf.org
sudo vim /etc/dhcpcd.conf
Add lines:
interface wlan0
    static ip_address=10.0.0.1/8
    nohook wpa_supplicant
```
3. Enable routing and IP masquerading (Skipped)
4. Configure the DHCP and DNS services for the wireless network
```sh
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.org
sudo vim /etc/dnsmasq.conf
Copy/Paste ./etc/dnsmasq.conf
```
5. Test configuration file
```sh
dnsmasq --test -C /etc/dnsmasq.conf
```
6. Restart service
```sh
sudo service dnsmasq restart
sudo service dnsmasq status
```
7. Ensure wireless operation
```sh
sudo rfkill unblock wlan
```
8. Configure the access point software
```sh
sudo vim /etc/hostapd/hostapd.conf
Copy/Paste ./etc/hostapd/hostapd.conf
```
9. Run your new wireless access point
```sh
sudo service hostapd restart
sudo service hostapd status
```

Debug mode
```sh
sudo systemctl stop hostapd
sudo hostapd -dd /etc/hostapd/hostapd.conf
```

---

## AirPlay audio player

- [Shairport Sync](https://github.com/mikebrady/shairport-sync)
- [Simple Installation Instructions](https://github.com/mikebrady/shairport-sync/blob/master/INSTALL.md)

1. Configure and Update
```sh
sudo apt update && sudo apt upgrade
```
2. Turn Off WiFi Power Management
```sh
sudo iwconfig wlan0 power off
```
3. Remove Old Copies
```sh
which shairport-sync
sudo rm /usr/local/bin/shairport-sync
sudo rm /etc/systemd/system/shairport-sync.service
sudo rm /lib/systemd/system/shairport-sync.service
sudo rm /etc/init.d/shairport-sync
sudo rm /etc/dbus-1/system.d/shairport-sync-dbus.conf
sudo rm /etc/dbus-1/system.d/shairport-sync-mpris.conf
```
4. Reboot after Cleaning Up
```sh
sudo reboot
```
5. Build and Install
```sh
sudo apt install --no-install-recommends build-essential git xmltoman autoconf automake libtool libpopt-dev libconfig-dev libasound2-dev avahi-daemon libavahi-client-dev libssl-dev libsoxr-dev
cd && mkdir -p src && cd src
git clone https://github.com/mikebrady/shairport-sync.git && cd shairport-sync
autoreconf -fi
./configure --sysconfdir=/etc --with-alsa --with-soxr --with-avahi --with-ssl=openssl --with-systemd
make
sudo make install
```
6. Configure
```sh
sudo vim /etc/shairport-sync.conf
// Sample Configuration File for Shairport Sync on a Raspberry Pi using the built-in audio DAC
general =
{
  volume_range_db = 60;
};

alsa =
{
  output_device = "hw:0";
  mixer_control_name = "PCM";
};
```
7. Start Shairport Sync on boot
```sh
sudo service shairport-sync enable
sudo service shairport-sync start
sudo service shairport-sync status
```

---

## [Shutdown via web](https://www.raspberrypi.org/forums/viewtopic.php?t=58802#p441115)

1. Install packages
```sh
sudo apt update && sudo apt install lighttpd php-cgi
sudo lighty-enable-mod fastcgi fastcgi-php
sudo service lighttpd force-reload
```
2. Edit the sudoers file
```sh
sudo visudo
# Add the following line below "pi ALL etc." and exit the visudo editor:
www-data ALL = NOPASSWD: /sbin/shutdown
(^s, ^x)
```
3. Create index.html
```sh
sudo vim /var/www/html/index.html
# Absolute minimum contents of the index.html file:
<html><a href="shutdown.php">Shutdown XCarlink NOW</a></html>
```
4. Create php function to shutdown
```sh
sudo vim /var/www/html/shutdown.php
# Absolute minimum contents of the shutdown.php file:
<?php system('sudo /sbin/shutdown -h now'); ?>
```
5. Test
```sh
(host) curl xcarlink.local/shutdown.php
```

---

## Debug

1. Services
```sh
sudo service owntone status
sudo service pulseaudio status
sudo service shairport-sync status
sudo service dbus status
sudo service dnsmasq status
sudo service hostapd status
```
2. Logs
```sh
tail -f /var/log/owntone.log
journalctl -u pulseaudio.service
```
3. Check that no pulseaudio in user mode is running
```sh
sudo ps -aux | grep pulse
```
4. Pulseaudio in debug mode
```sh
sudo service pulseaudio stop
sudo pulseaudio -vvvv
sudo service pulseaudio restart
```
