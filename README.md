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
debug.

## 2. Connect to the Pi

Now, I powered up the Pi and waited for it to boot - about a minute or so when the
green light is steady. The first step I took was to verify it had connected to
the Wifi by using the router admin. Once I saw the new device I tried to connect:
```
ssh pi@raspberrypi.local  # we can also use the IP address here
```

The default ssh password is `raspberry`.

If everything works, we're good to go! I noticed that the files created above
disappeared on the Pi, which means they are now set up in the OS' configuration.

## 3. Secure the Pi!

Obviously, I wanted to change the defaults to make it harder to access the Pi
for anyone on the network. First, I updated all the packages:
```
sudo apt-get update && sudo apt-get upgrade -y
```

Then, I ran the Pi's configuration tool to reset the hostname (raspberrypi) and
password (raspberry)
```
sudo raspi-config
```

I also used the interface options to enable ssh, but it was working even without that.

[Source for first 3 steps](https://www.losant.com/blog/getting-started-with-the-raspberry-pi-zero-w-without-a-monitor)

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
```
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

Apparently, I can use the following command to test volume, but I skipped this
part:
```
arecord -D dmic_sv -r 44100 -f S16_LE -c 2 --vumeter=stereo /dev/null
```

## 8. Install darkice

[Darkice](http://www.darkice.org/) is a live audio streamer that will record the
audio from the Turntable and send it to the streaming server (see icecast2
below).

Instead of taking on the task of compiling darkice, I used a pre-packaged deb
for darkice. My first attempt was to use a [newer version](https://debian.pkgs.org/10/debian-main-armhf/darkice_1.3-0.2_armhf.deb.html)
for Debian 10 (Buster) and Darkice v1.3, but that didn't work well. So I used
an older version that's been proven to work.

Both articles mentions compiling darkice to be used with mp3 support and all that.


## 9. Install icecast2

## 10. Configure darkice and have it autostart

## 11. Verify Icecast is working

## 12. Add stream to Sonos app




https://debian.pkgs.org/10/debian-main-armhf/darkice_1.3-0.2_armhf.deb.html

https://github.com/basdp/USB-Turntables-to-Sonos-with-RPi
https://www.instructables.com/Add-Aux-to-Sonos-Using-Raspberry-Pi/

http://code-injection.blogspot.com/2014/05/broadcasting-with-raspberry-pi.html
