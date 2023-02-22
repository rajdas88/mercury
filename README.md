# Mercury
## Connecting USB turntable to Sonos using Raspberry Pi

This contains step-by-step instructions to connect a USB turntable to Sonos
using the power of Raspberry Pi and Wifi. The main reason to do this is because
the alternative is a Sonos Port setting you back $450.

The basic system requires a modern turntable that has a USB output. This will be
used to stream audio to the Pi which in turn will cast it to Sonos.

### What I used
* Turntable - [AT-LP120XUSB](https://www.audio-technica.com/en-us/at-lp120xusb)
* [Raspberry Pi Zero W](https://www.raspberrypi.org/products/raspberry-pi-zero-w/)
* [Raspberry Pi power supply](https://www.adafruit.com/product/1995)
* [Micro SD card](https://www.amazon.com/gp/product/B073K14CVB/ref=ppx_yo_dt_b_asin_title_o02_s00?ie=UTF8&psc=1) - 16GB Class 10
  * [This is an authoritative list as well](https://elinux.org/RPi_SD_cards)
* [Raspberry Pi case](https://www.c4labs.com/product/zebra-zero-heatsink-case-raspberry-pi-zero-zero-w-color-options/)
* Micro USB OTG cable & cable to connect that to USB B

## 1. Prepare the Micro SD card

I used Windows 10 (as that laptop had a micro SD reader) for the next couple
steps, but the general gist is we need to set up the micro SD card with an OS for Raspberry Pi and set it up for headless access.

First, I used the [Raspberry Pi Imager](https://www.raspberrypi.org/software/)
to set up the micro SD card. I used the latest OS Lite image as I plan to only
connect via ssh.

Once the card is loaded with the OS, I needed to add 2 files.

1. In order to  enable ssh, I simply added a blank file called ssh into the root
directory of the device.

2. In order to enable wifi access, I created a new filed named
`wpa_supplicant.conf`. In this file, I added the following configuration:
  ```
  country=US
  ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
  update_config=1

  network={
      ssid="SSID of the Wifi (aka network name)"
      psk="The Wifi password"
      key_mgmt=WPA-PSK
  }
  ```

The above files are used by the Raspberry Pi to properly set up both ssh and
wireless access. These steps have to be done EXACTLY RIGHT or it's a pain to
debug. When done correctly, the `wpa_supplicant.conf` file will dissapear
from the card.

If the wifi information changes, recreate the `wpa_supplicant.conf` file and 
add to the micro SD card again. It will automatically overwrite the wifi 
config on the pi.

## 2. Connect to the Pi

Now, I powered up the Pi and waited for it to boot - about a minute or so when the
green light is steady. The first step I took was to verify it had connected to
the Wifi by using the router admin. Once I saw the new device I tried to connect:
```shell
ssh pi@raspberrypi.local  # we can also use the IP address here
```

The default ssh password is `raspberry`.

If everything works, we're good to go! I noticed that the files created above
disappeared on the Pi, which means they are now set up in the OS' configuration.

## 3. Secure the Pi!

Obviously, I wanted to change the defaults to make it harder to access the Pi
for anyone on the network. First, I updated all the packages:
```shell
sudo apt-get update && sudo apt-get upgrade -y
```

Then, I ran the Pi's configuration tool to reset the hostname (raspberrypi) and
password (raspberry)
```shell
sudo raspi-config
```

I also used the interface options to enable ssh, but it was working even without that.

[Source for first 3 steps](https://www.losant.com/blog/getting-started-with-the-raspberry-pi-zero-w-without-a-monitor)

## 3.1 Reboot the Pi when it loses wireless connection

Since the Pi was frequently unreachable, this script is to check if the Pi can
ping the router. If not, it will force restart the wifi. We store this script
in `/usr/local/bin/checkwifi.sh`
```shell
ping -c4 192.168.1.1 > /dev/null

if [ $? != 0 ]
then
  echo "No network connection, restarting wlan0"
  /sbin/ifdown 'wlan0'
  sleep 5
  /sbin/ifup --force 'wlan0'
fi
```

The more extreme alternative is to reboot the Pi:
```shell
ping -c4 192.168.1.1 > /dev/null

if [ $? != 0 ]
then
  sudo /sbin/shutdown -r now
fi
```

I changed permissions to make this script runnable `sudo chmod 775 /usr/local/bin/checkwifi.sh`.

Finally, I modified the crontab to run the above script every 10 minutes:
`*/10 * * * * /usr/bin/sudo -H /usr/local/bin/checkwifi.sh >> /dev/null 2>&1`

[Source](https://weworkweplay.com/play/rebooting-the-raspberry-pi-when-it-loses-wireless-connection-wifi/)

## 3.2 Log2Ram

As logs are frequently written to the Pi, this ends up causing a lot of writes
to the SD card which will lessen its' lifespan. This step writes logs to memory and
periodically writes those back to system logs.

First, I installed rsync which helps with performance
```shell
sudo apt install rsync
```
*Note: this was already installed*

Then I followed the install process:
```shell
echo "deb http://packages.azlux.fr/debian/ buster main" | sudo tee /etc/apt/sources.list.d/azlux.list
wget -qO - https://azlux.fr/repo.gpg.key | sudo apt-key add -
sudo apt update
sudo apt install log2ram
sudo reboot
```

After the reboot, to see if it's working:
```shell
$ df -h
…
log2ram          40M  532K   40M   2% /var/log
…

$ mount
…
log2ram on /var/log type tmpfs (rw,nosuid,nodev,noexec,relatime,size=40960k,mode=755)
…
```

[Source](https://github.com/azlux/log2ram#install)

## 3.3 Use SSH keys instead of password for Pi access

I copied the SSH key from my machine to the Pi with the following command:
```shell
ssh-copy-id <USERNAME>@<IP-ADDRESS>
```
This allows me to login without the password everytime.

[Source](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md)

## [Optional] 4. Put the Pi in a case

This step is pretty straightforward and useful to protect the Pi from the
elements. Instructions are found with the case, but
[linked here](https://www.c4labs.com/product/zebra-zero-heatsink-case-raspberry-pi-zero-zero-w-color-options/)
for posterity.

## [Debug] 5. SSH connectivity issues

While I was able to connect to the Pi immediately, after a few hours I wasn't
able to anymore. A day later it worked again without intervention. Some options
if this happens in the future:
1. DHCP reservation
2. Raspberry Pi Wifi power management
3. Heartbeat/ping cron job from Pi to keep the Wifi signal on

## 6. Connect the turntable to the Pi

Since my record player luckily already had USB out capabilities as well as a
preamp, I didn't have to do much here but connect this to the Pi's USB OTG port.
If the turntable doesn't have this capability, the
[Behringer U-Phono UFO202](https://www.amazon.com/gp/product/B002GHBYZ0?pldnSite=1)
should do the trick.

After, we can check the connection with:
```shell
pi@raspberrypi-mercury:~ $ arecord -l
**** List of CAPTURE Hardware Devices ****
card 1: CODEC [USB AUDIO  CODEC], device 0: USB Audio [USB Audio]
  Subdevices: 0/1
  Subdevice #0: subdevice #0

```

The card number (1 in my case) is needed for configuration later.

## 7. [Optional?] Volume adjustment issues

I followed this step directly from one of the tutorials and, in the future,
understand what it's doing and the necessity. Essentially, this is to give
software level volume control on the turntable. I created `/etc/asound.conf`
with the following:
```
pcm.dmic_hw {
    type hw
    card 1
    channels 2
    format S16_LE
}
pcm.dmic_mm {
    type mmap_emul
    slave.pcm dmic_hw
}
pcm.dmic_sv {
    type softvol
    slave.pcm dmic_hw
    control {
        name "Boost Capture Volume"
        card 1
    }
    min_dB -5.0
    max_dB 20.0
}
```
*Make sure the card number is right.*

Apparently, I can use the following command to test volume, but I skipped this
part:
```shell
arecord -D dmic_sv -r 44100 -f S16_LE -c 2 --vumeter=stereo /dev/null
```

[Source - step 4](https://github.com/basdp/USB-Turntables-to-Sonos-with-RPi)

## 8. Install darkice

[Darkice](http://www.darkice.org/) is a live audio streamer that will record the
audio from the Turntable and send it to the streaming server (see icecast2
below).

Instead of taking on the task of compiling darkice, I used a pre-packaged deb
for darkice. My first attempt was to use a [newer version](https://debian.pkgs.org/10/debian-main-armhf/darkice_1.3-0.2_armhf.deb.html)
for Debian 10 (Buster) and Darkice v1.3, but that didn't work well. So I used
an older version that's been proven to work and handles compiling darkice to be used with mp3 support.

I ran the following commands:
```shell
# Download the darkcice package
wget https://github.com/rajdas88/mercury/blob/master/darkice_1.0.1-999_mp3%2B1_armhf.deb
# Install other required packagess
sudo apt-get install libmp3lame0 libtwolame0
# Install the darkice package and fix dependency issues in next line
sudo dpkg -i darkice_1.0.1-999_mp3+1_armhf.deb
sudo apt-get install -f
```

## 9. Install icecast2

[Icecast](https://icecast.org/) is a streaming media server that will use the
mp3 created by darkice above and serve it to a URL that Sonos can pick up. The
installation is straightforward:
```shell
sudo apt-get install icecast2
```

When installing, there will be a few manual steps to change the hostname and
passwords. I kept the hostname as `localhost`, but change the passwords. The
password is used in the next step.

## 10. Configure darkice

I created `/etc/darkice.cfg` with the following contents:
```
[general]
duration        = 0      # duration in s, 0 forever
bufferSecs      = 1      # buffer, in seconds
reconnect       = yes    # reconnect if disconnected

[input]
device          = dmic_sv # Soundcard device for the audio input
sampleRate      = 44100   # sample rate 11025, 22050 or 44100
bitsPerSample   = 16      # bits
channel         = 2       # 2 = stereo

[icecast2-0]
bitrateMode     = cbr       # constant bit rate ('cbr' constant, 'abr' average)
#quality         = 1.0       # 1.0 is best quality (use only with vbr)
format          = mp3       # format. Choose 'vorbis' for OGG Vorbis
bitrate         = 320       # bitrate
server          = localhost # or IP
port            = 8000      # port for IceCast2 access
password        = <Password from step 9>    # source password for the IceCast2 server
mountPoint      = turntable.mp3  # mount point on the IceCast2 server .mp3 or .ogg
name            = Turntable
highpass        = 18
lowpass         = 20000
description	= Turntable
```

Some notes:
1. I had to change the password to what I had set in step 9.
2. The "quality" line is commented out with a # in front of it. It is used only if you set "bitrateMode = vbr" (variable bitrate). You can't have a quality value set when using cbr (constant bitrate) or the stream will stutter and skip.
[Source - step 13](https://www.instructables.com/Add-Aux-to-Sonos-Using-Raspberry-Pi/).
3. More configuration options can be found [here](http://manpages.ubuntu.com/manpages/hirsute/en/man5/darkice.cfg.5.html).

## 11. Start darkice and icecast and setup autostart

Next, I created a quick autostart script for darkice so it starts off everytime
I reboot. This uses a cron job. Create `darkice.sh` with the following contents:
```
#!/bin/bash
sudo /usr/bin/darkice -c /home/pi/darkice.cfg
```

Then I ran the following to give proper permissions and add the cron job:
```shell
sudo chmod 777 /home/pi/darkice.sh
crontab -e
```
And set up the cron table with:
```
@reboot sleep 30 && sudo /home/pi/darkice.sh
```
*I modified the wait a little longer to 30 seconds to be safe.*

Then I started icecast2:
```shell
sudo service icecast2 start
```

After all this is done, I rebooted the Pi and gave it 30 seconds to boot up:
```shell
sudo reboot
```

## 11. Verify Icecast is working

Next I verified that everything is working by visiting `http://<IP address of the Pi>:8000/`. I saw a mountpoint for turntable.mp3 and so it's all set!

I right clicked the M3U link to get the link address for the next step.

## 12. Add stream to Sonos app

Apparently, these steps can only be done in the Sonos Desktop app.
First, I added the stream to Sonos by following `Manage > Add Radio Station`.
The URL is from Icecase above and you can give the station whatever name.

Next, I added the TuneIn service. And then added the radio station,
`Radio by TuneIn > My Radio Stations` and found the station from above paragraph.

After, that I was set to play records from the turntable on to our Sonos speaker.

## Resources
1. https://github.com/basdp/USB-Turntables-to-Sonos-with-RPi
2. https://www.instructables.com/Add-Aux-to-Sonos-Using-Raspberry-Pi/
3. http://code-injection.blogspot.com/2014/05/broadcasting-with-raspberry-pi.html
