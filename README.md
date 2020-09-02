# PiZero-Bluetooth-Audio-Receiver
[![License: CC0-1.0](https://licensebuttons.net/l/zero/1.0/80x15.png)](http://creativecommons.org/publicdomain/zero/1.0/)
## How to turn your Raspberry Pi Zero W into a Bluetooth audio receiver.

### Objective
The goal of this how-to is to turn a Raspberry Pi Zero W into a headless Bluetooth audio receiver utilizing it's onboard Bluetooth module with packages available in the default repositories starting from a fresh Raspberry Pi OS Lite install. When complete your Pi Zero will function more or less like a simple Bluetooth speaker. It will be discoverable when no device is connected and accept all pairing requests via "Just Works" Secure Simple Pairing. Although several devices may be connected to it, only one may stream audio to it at a time.

### Expectations
It is assumed that you have started from a fully updated, unmodified, and fresh Raspberry Pi OS Lite install, have shell access, have very basic Linux knowledge and have some way of getting audio out of the Pi Zero. (The Pi Zero has no analog audio output) 

### Limitations
The Raspberry Pi Zero is not a very powerful device and the onboard Bluetooth module is not the greatest, it is known to have Wi-Fi coexistence issues (you may experience audio dropouts with wifi enabled), it is not suitable for low latency aplications and the package used in this how-to (bluealsa) only supports the SBC codec (no AAC, aptX*, or LDAC). If you're looking for an HD audiophile experience, this ain't it.

<i>*The dropout issue can sometimes be mitigated by disabling Wi-Fi and/or forcing turbo mode (the CPU will run full tilt all the time). Forcing turbo will however cause your Pi Zero to use more power and produce more heat. Under normal circumstances though heat is not a concern with a Pi Zero even with force turbo enabled.</i>

### On with it then...

<b>Install the packages needed:</b>

 ```sudo apt install -y --no-install-recommends git alsa-base alsa-utils bluealsa bluez-tools```


<b>Clone the repo and cd into the repo's folder:</b>

```git clone https://github.com/JasonLG1979/PiZero-Bluetooth-Audio-Receiver.git && cd PiZero-Bluetooth-Audio-Receiver```


<b>Create the folder that will contain our sounds:</b>

```sudo mkdir -p /usr/local/share/sounds/__custom```


<b>Copy the sounds to our new folder:</b>

```sudo cp device-added.wav /usr/local/share/sounds/__custom/```

```sudo cp device-removed.wav /usr/local/share/sounds/__custom/```

```sudo cp discoverable.wav /usr/local/share/sounds/__custom/```


<b>cd back out of the folder and delete it (if you don't plan on contributing to the repo):</b>

```cd && rm -rf PiZero-Bluetooth-Audio-Receiver```


<b>Create a bluealsa group:</b>

```sudo addgroup --system bluealsa```


<b>Create an unprivileged bluealsa system user in the bluealsa group:</b>

```sudo adduser --system --disabled-password --disabled-login --no-create-home --ingroup bluealsa bluealsa```


<b>Add the bluealsa user to the bluetooth group:</b>

```sudo adduser bluealsa bluetooth```


<b>Add the bluealsa user to the audio group:</b>

```sudo adduser bluealsa audio```

<b>Create a bt-agent group:</b>

```sudo addgroup --system bt-agent```


<b>Create an unprivileged bt-agent system user in the bt-agent group:</b>

```sudo adduser --system --disabled-password --disabled-login --no-create-home --ingroup bt-agent bt-agent```

<b>Add the bt-agent user to the Bluetooth group:</b>

```sudo adduser bt-agent bluetooth```


<b>Edit ```/etc/bluetooth/main.conf``` to disable the discoverable timeout and change our device class to "HiFi Audio Device":</b>

```sudo nano /etc/bluetooth/main.conf```
 

Change:

```#Class = 0x000100```

```#DiscoverableTimeout = 0```

```#FastConnectable = false``` 

To:

```Class = 0x200428```

```DiscoverableTimeout = 0```

```FastConnectable = true``` 


Save and exit nano (ctrl+x, y, enter)


<b>Create an override for the bluetooth.service that disables unneeded plugins:</b>

```sudo systemctl edit bluetooth.service```

Paste this into the file:
```
[Service]
ExecStart=
ExecStart=/usr/lib/bluetooth/bluetoothd  --noplugin=sap,network,hog,health,midi
```
Save and exit nano (ctrl+x, y, enter)


<b>Reload the systemd daemon:</b>

```sudo systemctl daemon-reload```


<b>Edit ```/etc/dbus-1/system.d/bluealsa.conf``` to allow our unprivileged bluealsa system user to run the bluealsa daemon:</b>

```sudo nano /etc/dbus-1/system.d/bluealsa.conf```

Change:

```<policy user="root">```

To:

```<policy user="bluealsa">```

Save and exit nano (ctrl+x, y, enter)


<b>Override the bluealsa service:</b>

```sudo systemctl edit --full bluealsa.service```

Delete everything and paste this into it:

```
[Unit]
Description=BluezALSA proxy
Requires=bluetooth.service
After=bluetooth.service

[Service]
Type=dbus
BusName=org.bluealsa
User=bluealsa
Group=bluealsa
NoNewPrivileges=true
ExecStart=/usr/bin/bluealsa -i hci0 -p a2dp-sink --a2dp-force-audio-cd
Restart=on-failure
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
RemoveIPC=true
RestrictAddressFamilies=AF_UNIX AF_BLUETOOTH

[Install]
WantedBy=multi-user.target
```
Save and exit nano (ctrl+x, y, enter)


<b>Reload the systemd daemon:</b>

```sudo systemctl daemon-reload```

<b>Enable the bluealsa.service:</b>

```sudo systemctl enable bluealsa.service```


<b>Create the bluealsa-aplay service:</b>

```sudo nano /etc/systemd/system/bluealsa-aplay.service```

Paste this into the file:
```
[Unit]
Description=Bluealsa audio player
Requires=bluealsa.service
After=bluealsa.service

[Service]
Type=simple
ExecStart=/usr/bin/bluealsa-aplay --profile-a2dp --single-audio --pcm-buffer-time=1000000 00:00:00:00:00:00
Restart=on-failure
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
RemoveIPC=true
RestrictAddressFamilies=AF_UNIX
User=bluealsa
Group=audio
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```
Save and exit nano (ctrl+x, y, enter)


<b>Enable the bluealsa-aplay service:</b>

```sudo systemctl enable bluealsa-aplay.service```


<b>Create the bt-agent service to enable "Just Works" Bluetooth pairing:</b>

```sudo nano /etc/systemd/system/bt-agent.service```

Paste this into the file:
```
[Unit]
Description=Bluetooth Agent
Requires=bluealsa-aplay.service
After=bluealsa-aplay.service

[Service]
Type=simple
User=bt-agent
Group=bt-agent
ExecStart=/usr/bin/bt-agent --capability=NoInputNoOutput
Restart=always
NoNewPrivileges=true
KillSignal=SIGUSR1
Restart=on-failure
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
RemoveIPC=true
RestrictAddressFamilies=AF_UNIX

[Install]
WantedBy=multi-user.target

```
Save and exit nano (ctrl+x, y, enter)


<b>Enable the bt-agent service:</b>

```sudo systemctl enable bt-agent.service```


<b>Create the bt-discovery service to enabe discoverability at startup and be able to toggle it on and off with systemd:</b>

```sudo nano /etc/systemd/system/bt-discovery.service```

Paste this into the file:
```
[Unit]
Description=Toggle bluetooth discoverable
Requires=bt-agent.service
After=bt-agent.service

[Service]
Type=oneshot
ExecStartPre=/usr/bin/aplay -q /usr/local/share/sounds/__custom/discoverable.wav
ExecStart=/usr/bin/bluetoothctl discoverable on
ExecStop=/usr/bin/bluetoothctl discoverable off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

<b>Enable the bt-discovery service:</b>

```sudo systemctl enable bt-discovery.service```


<b>Nuke the useless bthelper service:</b>

```sudo systemctl disable bthelper@.service```

```sudo systemctl mask bthelper@.service```

```sudo rm /lib/systemd/system/bthelper@.service```

```sudo systemctl daemon-reload```

```sudo systemctl reset-failed```

<b>Create a udev script so our Pi Zero is only discoverable if no devices are connected:</b>

```sudo nano /usr/local/bin/bluetooth-udev```

Paste this into the file:
```
#!/bin/bash
if [[ ! $NAME =~ ^\"([0-9A-F]{2}[:-]){5}([0-9A-F]{2})\"$ ]]; then exit 0; fi
action=$(expr "$ACTION" : "\([a-zA-Z]\+\).*")
if [ "$action" = "add" ]; then
    aplay -q /usr/local/share/sounds/__custom/device-added.wav
    systemctl stop bt-discovery.service
fi
if [ "$action" = "remove" ]; then
    aplay -q /usr/local/share/sounds/__custom/device-removed.wav
    deviceinfo=$(bluetoothctl info)
    if [ "$deviceinfo" = "Missing device address argument" ]; then
        systemctl start bt-discovery.service
    fi
fi
```
Save and exit nano (ctrl+x, y, enter)


<b>Set the proper permissions for the script:</b>

```sudo chmod 755 /usr/local/bin/bluetooth-udev```


<b>Create a udev rule to call the scrpit:</b>

```sudo nano /etc/udev/rules.d/99-bluetooth-udev.rules```


Paste this into the file:
```
SUBSYSTEM=="input", GROUP="input", MODE="0660"
KERNEL=="input[0-9]*", RUN+="/usr/local/bin/bluetooth-udev"
```
Save and exit nano (ctrl+x, y, enter)

<b>Reboot:</b>

```sudo reboot```

Your Pi Zero should be discoverable and show up to other devices as a Bluetooth audio receiver. It's name will be whatever your Pi Zero's hostname is.


### Audio Setup

Audio will play out the default output device. It will be 16 bit 44.1 khz. Upsampling it to a value that is not a multiple of 44.1 khz will degrade the sound quality and cost you valuable CPU cycles up to the point of causing audio dropouts. If you are using a DAC hat follow the manufacturer's documentation to setup your DAC hat as the default output device. If you are using a USB DAC you can use ```aplay -l``` to find your card.

An example output of ```aplay -l``` is here:
```
**** List of PLAYBACK Hardware Devices ****
card 0: Headphones [bcm2835 Headphones], device 0: bcm2835 Headphones [bcm2835 Headphones]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
card 1: DAC [USB Audio DAC], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

Edit ```/usr/share/alsa/alsa.conf```:

```sudo nano /usr/share/alsa/alsa.conf```

Change:

```defaults.ctl.card 0```

```defaults.pcm.card 0```

To:

```defaults.ctl.card <card #>```

```defaults.pcm.card <card #>```

Replace all instances of:

```<card #>```


With whatever card you want to use from the output of ```aplay -l```.


While you're at it if your card support 44.1 khz:

Change:

```defaults.pcm.dmix.rate 48000```

To:

```defaults.pcm.dmix.rate 44100```

To prevent unnecessary upsampling.


If you're an audio purist, your DAC supports 16bit 44.1 khz, doesn't sound like butt at 0dB, and you only plan on using your Pi Zero as a Bluetooth audio receiver with no other audio programs you can skip all software mixing, resampling and volume control and expose the raw audio device. (this is my preferred configuration)


Edit ```/etc/asound.conf```:

```sudo nano  /etc/asound.conf```


```
defaults.pcm.card <card #>
defaults.ctl.card <card #>

pcm.!default {
    type hw
    card <card #>
}

ctl.!default {
    type hw
    card <card #>
}

```

Replace all instances of:

```<card #>```


With whatever card you want to use from the output of ```aplay -l```.


So in the above example using a USB DAC and the USB DAC being card 1 it would look like this:

```
defaults.pcm.card 1
defaults.ctl.card 1

pcm.!default {
    type hw
    card 1
}

ctl.!default {
    type hw
    card 1
}
```

Save and exit nano (ctrl+x, y, enter)


Reboot and enjoy!!!:

```sudo reboot```


### Audio Troubleshooting

<i>I've got everything working but the audio sounds compressed and/or distorted.</i>

Not all DACs are created equal. Up to the point of diminishing returns (maybe $100?) you get pretty much what you pay for. Welcome to the worlds of [jitter](https://en.wikipedia.org/wiki/Jitter) and [intersample peaks](https://www.productionmusiclive.com/blogs/news/mastering-tip-what-are-inter-sample-peaks-why-they-matter) both of which cause distortion (most consumer grade DACs have zero volume headroom, so at full digital volume any intersample peaks above 0.0dB will clip and distort) No amount of tweaking is going to make a $5 USB DAC or $15 DAC HAT sound great, and there's really not anything that can be done about jitter (except buy a better DAC) but we can maybe make the DAC tolerable by giving it a little bit of digital headroom to help prevent some of the intersample clipping and see if that helps.

To do this we'll use the softvol ALSA plugin.

<b>Edit ```/etc/asound.conf```</b>.

```sudo nano /etc/asound.conf```

If you don't care about software mixing and just want a little headroom paste this into the file:

```
defaults.pcm.card <card #>
defaults.ctl.card <card #>

pcm.!default {
    type softvol
    slave.pcm "hw:<card #>"
    control {
        name "Softvol"
        card <card #>
    }
    max_dB -3.0
}
```

Or if you do want software mixing paste this into the file:

```
defaults.pcm.card <card #>
defaults.ctl.card <card #>

pcm.dmixer {
    type dmix
    ipc_key 1024
    ipc_perm 0666
    slave.pcm "hw:<card #>"
    slave {
        rate 44100
        format S16_LE
    }
    bindings {
        0 0
        1 1
    }
}

ctl.dmixer {
    type hw
    card <card #>
}

pcm.softvol {
    type softvol
    slave.pcm "dmixer"
    control {
      name "Softvol"
      card <card #>
    }
    max_dB -3.0
}

pcm.!default {
    type plug
    slave.pcm "softvol"
}

```
Replace all instances of:

```<card #>```


With whatever card you want to use from the output of ```aplay -l```.


Save and exit nano (ctrl+x, y, enter)


<b>Reboot</b>

```sudo reboot```


<b>Stream some audio to your Pi Zero and see if that helped at all</b>


<i>I'm trying to change the volume directly on the Pi Zero and nothing happens.</i>

Your DAC doesn't have hardware volume control, or at least it doesn't work. Follow the above section and enable software volume control, remove the ```max_dB -3.0``` line if you don't need additional volume headroom.


<i>I can hear stuff but the volume is really low.</i>

Turn the volume up on the Pi Zero with ```alsamixer```.

```alsamixer -c<card #>```

Replace:

```<card #>```


With whatever card you want to control from the output of ```aplay -l```.

alsamixer is pretty self explanatory...


### Bonus Points!!!

<b>!!!WARNING!!!</b>

These overclock setting WILL <b>void your warranty</b> and eat your cat(s). I am not responsible for any damages, demon possessions or unwanted pregnancies.


That being said I've had good results with the following to eliminate audio dropouts with just a small heatsink and a case with no active cooling or ventilation. (Temps never got over 57c and it never throttled during a 60 min stress test, your mileage may vary of course.)

```
boot_delay=1
force_turbo=1
arm_freq=1100
over_voltage=8
sdram_over_voltage=8
core_freq=500
sdram_freq=500
```

### Credits

By:

[JasonLG1979](https://github.com/JasonLG1979)

Inspired greatly by the [install-bluetooth script](https://github.com/nicokaiser/rpi-audio-receiver/blob/master/install-bluetooth.sh) by [nicokaiser](https://github.com/nicokaiser) and others in the [rpi-audio-receiver](https://github.com/nicokaiser/rpi-audio-receiver) repo.

Bluealsa systemd integration adapted from the [Arkq/bluez-alsa wiki](https://github.com/Arkq/bluez-alsa/wiki/Systemd-integration)
