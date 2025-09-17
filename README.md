# CVLC Network Video Streamer

## Introduction
This is a simple Raspberry Pi-based video streamer, using `cvlc`, the console variant of VLC.

The streamer needs no desktop - just DRM video access and ALSA access to the HDMI output.

The streamer starts on boot and then tries to load a HTTP video source - file or playlist.

The instructions include infrared remote control that make it a basic set-top box. 

All that's needed is:
- Raspberry Pi with power and HDMI cable
- SD card with Raspberry Pi Lite image (no need for desktop)
- Raspberry Pi Lite configured with networking
- (Optional) Infrared Receiver (in this we use a TSOP hardwired to GPIO)

This repo's contents:
- cvlc.lircrc - config file outlining how to control cvlc with an infrared remote
- cvlc.server - script to (re)start, stop cvlc with valid URL checking
- irexec.lircrc - configuration for powering down the Raspberry PI via an infrared remote
- *.conf - a file containing a infrared remote LIRC profile (can be made in this guide)

## Install CVLC

First, install VLC minimally as no desktop is needed:
```
sudo apt update
sudo apt upgrade
sudo apt install vlc --no-install-recommends
```

## Infrared Controller

It is possible to wire a TSOP-package IR receiver to the GPIO on 3.3v (Pin 1), Ground (Pin 6) and GPIO 18 (Pin 12):

```
sudo sh -c "echo dtoverlay=gpio-ir,gpio_pin=18 >> /boot/firmware/config.txt"
sudo reboot
```

To test and configure the infrared receiver, install LIRC minimally:
```
sudo apt install lirc --no-install-recommends
```

Then to test the remote in a raw mode to see if it's working:
```
mode2 -d /dev/lirc0
```

To configure a new remote, load up the name of the keys and use irrecord to record:
```
irrecord --list-namespace
irrecord -d /dev/lirc0
```

Once an infrared remote config is created it can then be installed. Swap the driver to "default" if using a GPIO receiver, restart and then test using `irw`:
```
sudo cp Samsung_AA59-00741A.lircd.conf /etc/lirc/lircd.conf.d/
sudo sed -i 's/devinput/default/g' /etc/lirc/lirc_options.conf 
sudo systemctl enable lircd
sudo systemctl start lircd
irw
```

If all is well, install a global irexec for `KEY_POWER` to shut down the Raspberry Pi:
```
sudo cp irexec.lircrc /etc/lirc/
sudo systemctl enable irexec
```

## CVLC Testing
To test CVLC, test with a valid URL. Just make sure your user has `video` and `audio` group membership:
```
cvlc --control lirc --lirc-file cvlc.lircrc --vout drm_vout --aout any http://hostname:1234/path/to/video/source
```

The cvlc.lircrc file contains bindings for KEY_UP / KEY_DOWN / KEY_POWER for channel navigation and power. 

## CVLC Booting

If this works, make the CVLC_URL static:
```
sudo sh -c "echo CVLC_URL=http://hostname:1234/path/to/video/source > /etc/default/cvlc"
```

Finally, for it all to boot, create a dedicated user, install configs and enable the service:
```
sudo useradd -M -d /nonexistent -g video -G audio -s /usr/sbin/nologin cvlc
sudo cp cvlc.lircrc /etc/lirc/
sudo cp cvlc.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable cvlc.service
sudo systemctl start cvlc.service
```
