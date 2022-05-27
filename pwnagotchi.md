# Making a Pwnagotchi

### Materials

I have listed all the parts I used to make my pwnagotchis below. Amazon links are provided for the items I bought, except the Raspberry Pi, which links to the Raspberry Pi site directly.

1. [Raspberry Pi Zero W](https://www.raspberrypi.com/products/raspberry-pi-zero-w/)
2. [Micro SD Card](https://www.amazon.com/dp/B09B1F9L52?ref=ppx_yo2ov_dt_b_product_details&th=1)
3. [Waveshare eInk Display Hat v3](https://www.amazon.com/dp/B07Z1WYRQH?psc=1&ref=ppx_yo2ov_dt_b_product_details)
4. [DS3231 RTC Module](https://www.amazon.com/dp/B01N1LZSK3?psc=1&ref=ppx_yo2ov_dt_b_product_details)
5. USB data cable

##### Note: All steps noted below were performed from a linux machine

## Installation

I spent several iterations trying to get my pwnagotchi to work with v1.5.5 directly, but this version has a [known issue](https://github.com/evilsocket/pwnagotchi/issues/992) that bugs the AI. I found and tried various fixes, and did manage to get the AI working, but the advertisement packets (how your pwnagotchi tells peers it exists) were suspisciously missing. Rather than troubleshoot a new issue, I chose to revert to the [v1.5.3 release](https://github.com/evilsocket/pwnagotchi/releases) and upgrade to v1.5.5 over an internet tether. This has yielded a pwnagotchi that has both AI and peer functionality.

### Getting the Image

Download the [v1.5.3 release](https://github.com/evilsocket/pwnagotchi/releases). Once downloaded, I decompressed the file with 7zip

```bash
7z e /path/to/pwnagotchi-raspbian-lite-v1.5.3.zip
```

### Flashing

Once I had the image file, I plugged the micro SD card into my linux machine with a USB adapter and flashed the image with the `dd` command, adding the `pv` permutation to show a better progress bar. (Normal `dd` can be used with `status=progress`)

```bash
sudo pv -ptre /path/to/pwnagotchi-raspbian-lite-v1.5.3.img | sudo dd of=/path/to/SD_card bs=1M iflag=fullblock oflag=direct,sync
```

### Configuration

Next, I needed to add the initial configuration. To do so, I navigated to the `/boot` partition of the newly flashed card and create a file called `config.toml`.

The [recommended config](https://pwnagotchi.ai/configuration/) is great to start with, but after setting up my pwnagotchi several times, I knew which settings I would want to change immediately, and have added them to my initial config.

The most important addition I made was the use of the `auto-update` plugin, which is how I'll be updating to Pwnagotchi v1.5.5.

##### Contents of `config.toml`

```toml
[main]
name = "pwnagotchi" # Name your pwnagotchi here
lang = "en"
whitelist = [
  "YOUR_HOME_NETWORK",
  "aa:bb:cc:dd:ee:ff"
]

[ui.display]
enabled = true
type = "waveshare_2"
color = "black"

[personality] # Disabling both options makes the pwnagotchi passive
deauth = false
associate = false

[ui.web]
enabled = true
username = "changeme"
password = "changeme"

[main.plugins.grid]
enabled = true
report = true
exclude = [
  "YOUR_HOME_NETWORK",
  "aa:bb:cc:dd:ee:ff"
]

[main.plugins.memtemp]
enabled = true
scale = "celsius"
orientation = "horizontal"

[main.plugins.auto-update] # Update when internet connected
enabled = true
install = true
interval = 1
```

This file will install to `/etc/pwnagotchi/config.toml`, and can be edited there later.

### First Boot and SSH

To update your pwnagotchi, connect it to your pc with a micro USB data cable, using the data port of the RPi Zero. First boot will take several minutes, as noted on the official website. Once connected, your pc should detect it as a wired connection or usb interface in the network settings. I used the following network settings, per the official guide.

| Parameter | Value         |
| --------- | ------------- |
| Address   | 10.0.0.1      |
| Netmask   | 255.255.255.0 |
| Gateway   | 8.8.8.8       |

These settings may take a moment to take hold. Once they do, you should be able to SSH into your pwnagotchi (once it boots) using

```bash
ssh pi@10.0.0.2
```

The default password is "raspberry", and should be changed.

### Internet Connection

Once connected, I needed to set up my host machine and the Raspberry Pi Zero for NAT forwarding. I recommend using the [script](https://pwnagotchi.ai/configuration/#host-connection-sharing) provided by pwnagotchi, developing your own if you have any issues. I ended up writing my own so that I could learn the process better and make a few tweaks that helped on my system.

##### Contents of `internet_share.sh`

```bash
#!/usr/bin/env bash
set -e

ETH0=eth0 # Host machine interface
ETH1=usb0 # Raspberry Pi Zero interface
ETH1_IP=10.0.0.1
ETH1_NET=10.0.0.0/24

# Clear IP tables for RPi device on host machine
ip addr flush dev "$ETH1"

# Configure static ip for RPi
ip addr add "$ETH1_IP/24" dev "$ETH1"

# Allow forwarded packets
iptables -A FORWARD -o "$ETH0" -i "$ETH1" -s "$ETH1_NET" -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Setup NAT
iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o "$ETH0" -j MASQUERADE

# Enable IP forwarding on host machine
sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"

```

On my host machine running Ubuntu, I also had to edit my `/etc/sysctl.conf` file to allow forwarding. Check your own distro for specific instructions.

##### In `/etc/sysctl.conf`, uncomment this line

```
net.ipv4.ip_forward=1
```

Now all I had to do was change the nameserver of the pi over ssh. This line is located in `/etc/resolv.conf`

##### Updated nameserver on the RPi

```
nameserver 8.8.8.8
```

Once this is done, ping a website of your choice to check the connection. If that is successful, check the Pwnagotchi's logs with

```
tail -f /var/log/pwnagotchi
```

### Updating

Because I enabled the auto-update plugin, the pi should begin to update as soon as it recognizes internet connectivity. This update process will be reflected in the logs, and will end with a reboot. After this reboot, you may need to reconfigure your network settings to connect to the RPi again, unless you have performed [this](https://pwnagotchi.ai/community/#static-rdnis-gadget-to-avoid-reconfiguration-everytime-you-plug-it-to-the-computer) community hack.

### Waveshare eInk v3

When I ordered my screen from Amazon, I could only find the v3 model. Luckily, [this](https://github.com/ikornaselur/pwnagotchi) repository has all of the code required to enable the v3 hat. I went to [this](https://github.com/ikornaselur/pwnagotchi/commit/91e95cede0b17aeb1a09f13930869baa245a5616) commit and manually added the files to the RPi over ssh rather than cloning over the entire pwnagotchi dist.

Once all of these files are supplemented to your pwnagotchi, edit `/etc/pwnagotchi/config.toml`

```
[ui.display]
enabled = true
type = "waveshare_3" # Update this line
color = "black"
```

Perform a reboot with the v3 hat attached, and after a minute or two, you should see the screen (and your new friend) coming to life.

### Adding a Real Time Clock (RTC)

While not necessary, I thought it would be nice if my pwnagotchi could keep time. I decided to use a DS3231 module, obtained from Amazon. For my modules, I simply needed to desolder and remove a factory installed header, and solder on my own jump wires for `vcc`, `clock`, `data`, and `ground`. Once that was done, I connected those wires to the corresponding pins on the RPi (pinout chart [here](https://pi4j.com/1.2/images/j8header-zero.png)).

To facilitate cleanliness, I attached the DS3231's lead wires to the bottom of the RPi, simply reflowing the solder pads for the RPi's header and attaching the appropriate leads.

Once the RTC was attached, I used

```bash
sudo raspi-config
```

to access the RPi's configuration menu and enable I2C. Once enabled, I followed the directions [here](https://learn.adafruit.com/adding-a-real-time-clock-to-raspberry-pi/set-rtc-time) to set up the RTC.

First, I edited `/boot/config.txt` on the RPi to include

```
dtoverlay=i2c-rtc,ds3231
```

Then I rebooted the system, reconnected via ssh, and ran

```bash
i2cdetect -y 1
```

You should see "UU" on one of the channels.

Next, I disabled the fake hardware clock

```bash
sudo apt-get -y remove fake-hwclock
sudo update-rc.d -f fake-hwclock remove
sudo systemctl disable fake-hwclock
```

Then I edited `/lib/udev/hwclock-set` and commented out the following sections:

```
#if [ -e /run/systemd/system ] ; then
# exit 0
#fi
```

```
#/sbin/hwclock --rtc=$dev --systz --badyear
```

```
#/sbin/hwclock --rtc=$dev --systz
```

I used

```bash
sudo hwclock -r
```

to read the time from the RTC. It was wrong, as was expected. I set my locale with

```bash
timedatectl set-timezone <your_timezone_here>
```

and ran the script to tether internet through my host machine again. I waited for NTP to take hold, checking the system time with

```bash
date
```

Once it settled in, I wrote this time to the hardware clock with

```bash
sudo hwclock -w
```

you can check that it worked by issuing the same command with the `-r` flag again. As long as the time takes hold, you should be good to go. This can be verified by unpluggin your RPi, reconnecting through SSH (no internet required this time) and checking the device time. If it matches what your host machine says, you are good to go!

### Final Thoughts

Pwnagotchi is a really cool project, and I have enjoyed building a few of them over the last week or so. It's peaked my interest in packet sniffing and hash cracking all over again, and as the official website says, "It's cute as f---". I would recommend building one (or more) of these cantankerous little bastards to anyone with an interest in learning how wireless networking works, wanting to learn a bit more about linux (I learned how to NAT forward for the first time with this project), or anybody who just wants a project that isn't too difficult, but has hardware and software components to tinker with.
