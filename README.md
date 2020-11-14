# PiZero-Bluetooth-Audio-Receiver
[![License: CC0-1.0](https://licensebuttons.net/l/zero/1.0/80x15.png)](http://creativecommons.org/publicdomain/zero/1.0/)
## How to turn your Raspberry Pi Zero W into a Bluetooth audio receiver.

### Objective
The goal of this how-to is to turn a Raspberry Pi Zero W into a headless Bluetooth audio receiver utilizing it's onboard Bluetooth module with packages available in the default repositories starting from a fresh Raspberry Pi OS Lite install. When complete your Pi Zero will function more or less like a simple Bluetooth speaker. It will be discoverable when no device is connected and accept all pairing requests via "Just Works" Secure Simple Pairing. Although several devices may be connected to it, only one may stream audio to it at a time.

### Expectations
It is assumed that you have started from a fully updated, unmodified, and fresh Raspberry Pi OS Lite install, have shell access, have very basic Linux knowledge and have some way of getting audio out of the Pi Zero. (The Pi Zero has no analog audio output) 

### Limitations
The Raspberry Pi Zero is not a very powerful device and the onboard Bluetooth module is not the greatest, it is known to have Wi-Fi coexistence issues (you may experience audio dropouts with wifi enabled), it is not suitable for low latency aplications and the package used in this how-to (bluealsa) only supports the SBC codec (no AAC, aptX*, or LDAC). If you're looking for an HD audiophile experience, this ain't it.

_*The dropout issue can sometimes be mitigated by disabling Wi-Fi and/or forcing turbo mode (the CPU will run full tilt all the time). Forcing turbo will however cause your Pi Zero to use more power and produce more heat. Under normal circumstances though heat is not a concern with a Pi Zero even with force turbo enabled._

### On with it then...

**Install the packages needed:**

 ```sudo apt install -y --no-install-recommends git bluealsa bluez-tools```


**Clone the repo and cd into the repo's folder:**

```git clone https://github.com/JasonLG1979/PiZero-Bluetooth-Audio-Receiver.git && cd PiZero-Bluetooth-Audio-Receiver```


**Create the folder that will contain our sounds:**

```sudo mkdir -p /usr/local/share/sounds/__custom```


**Copy the sounds to our new folder:**

```sudo cp device-added.wav /usr/local/share/sounds/__custom/```

```sudo cp device-removed.wav /usr/local/share/sounds/__custom/```

```sudo cp discoverable.wav /usr/local/share/sounds/__custom/```


**cd back out of the folder and delete it (if you don't plan on contributing to the repo):**

```cd && rm -rf PiZero-Bluetooth-Audio-Receiver```


**Create a bluealsa group:**

```sudo addgroup --system bluealsa```


**Create an unprivileged bluealsa system user in the bluealsa group:**

```sudo adduser --system --disabled-password --disabled-login --no-create-home --ingroup bluealsa bluealsa```


**Add the bluealsa user to the bluetooth group:**

```sudo adduser bluealsa bluetooth```


**Add the bluealsa user to the audio group:**

```sudo adduser bluealsa audio```

**Create a bt-agent group:**

```sudo addgroup --system bt-agent```


**Create an unprivileged bt-agent system user in the bt-agent group:**

```sudo adduser --system --disabled-password --disabled-login --no-create-home --ingroup bt-agent bt-agent```

**Add the bt-agent user to the Bluetooth group:**

```sudo adduser bt-agent bluetooth```


**Edit ```/etc/bluetooth/main.conf``` to disable the discoverable timeout and change our device class to "HiFi Audio Device":**

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


**Create an override for the bluetooth.service that disables unneeded plugins:**

```sudo systemctl edit bluetooth.service```

Paste this into the file:
```
[Service]
ExecStart=
ExecStart=/usr/lib/bluetooth/bluetoothd  --noplugin=sap,network,hog,health,midi
```
Save and exit nano (ctrl+x, y, enter)


**Reload the systemd daemon:**

```sudo systemctl daemon-reload```


**Edit ```/etc/dbus-1/system.d/bluealsa.conf``` to allow our unprivileged bluealsa system user to run the bluealsa daemon:**

```sudo nano /etc/dbus-1/system.d/bluealsa.conf```

Change:

```<policy user="root">```

To:

```<policy user="bluealsa">```

Save and exit nano (ctrl+x, y, enter)


**Override the bluealsa service:**

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
ExecStart=/usr/bin/bluealsa -i hci0 -p a2dp-sink
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


**Reload the systemd daemon:**

```sudo systemctl daemon-reload```

**Enable the bluealsa.service:**

```sudo systemctl enable bluealsa.service```


**Create the bluealsa-aplay service:**

```sudo nano /etc/systemd/system/bluealsa-aplay.service```

Paste this into the file:
```
[Unit]
Description=Bluealsa audio player
Requires=bluealsa.service
After=bluealsa.service

[Service]
Type=simple
ExecStart=/usr/bin/bluealsa-aplay --profile-a2dp --single-audio 00:00:00:00:00:00
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


**Enable the bluealsa-aplay service:**

```sudo systemctl enable bluealsa-aplay.service```


**Create the bt-agent service to enable "Just Works" Bluetooth pairing:**

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


**Enable the bt-agent service:**

```sudo systemctl enable bt-agent.service```


**Create the bt-discovery service to enabe discoverability at startup and be able to toggle it on and off with systemd:**

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

**Enable the bt-discovery service:**

```sudo systemctl enable bt-discovery.service```


**Nuke the useless bthelper service:**

```sudo systemctl disable bthelper@.service```

```sudo systemctl mask bthelper@.service```

```sudo rm /lib/systemd/system/bthelper@.service```

```sudo systemctl daemon-reload```

```sudo systemctl reset-failed```

**Create a udev script so our Pi Zero is only discoverable if no devices are connected:**

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


**Set the proper permissions for the script:**

```sudo chmod 755 /usr/local/bin/bluetooth-udev```


**Create a udev rule to call the scrpit:**

```sudo nano /etc/udev/rules.d/99-bluetooth-udev.rules```


Paste this into the file:
```
SUBSYSTEM=="input", GROUP="input", MODE="0660"
KERNEL=="input[0-9]*", RUN+="/usr/local/bin/bluetooth-udev"
```
Save and exit nano (ctrl+x, y, enter)

**Reboot:**

```sudo reboot```

Your Pi Zero should be discoverable and show up to other devices as a Bluetooth audio receiver. It's name will be whatever your Pi Zero's hostname is.


### Audio Setup

**Find your card**

   `aplay -l`

Should output something like this:
```
**** List of PLAYBACK Hardware Devices ****
card 0: sndrpihifiberry [snd_rpi_hifiberry_dac], device 0: HifiBerry DAC HiFi pcm5102a-hifi-0 [HifiBerry DAC HiFi pcm5102a-hifi-0]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
```

**Find your card's supported format(s) and sampling rate(s)**

While no other audio is being played:
```
aplay -Dhw:<card #>,<device #> --dump-hw-params /usr/share/sounds/alsa/Front_Right.wav
```
For example:
```
aplay -Dhw:0,0 --dump-hw-params /usr/share/sounds/alsa/Front_Right.wav
```
The output should look something like this if the card and device combination is correct:
```
ACCESS:  MMAP_INTERLEAVED RW_INTERLEAVED
FORMAT:  S16_LE S24_LE S32_LE
SUBFORMAT:  STD
SAMPLE_BITS: [16 32]
FRAME_BITS: [32 64]
CHANNELS: 2
RATE: [8000 192000]
...
```
The above card supports formats S16_LE, S24_LE, and S32_LE, at up to a sampling rate of 192000.

**Check to see if your sound card has hardware volume controls**
```
amixer -c<card #> scontrols
```
For example:
```
amixer -c0 scontrols
```
The output should look something like this if your card has hardware volume control:
```
Simple mixer control 'PCM',0
...
```
**So by now we should know the card # the device # the supported formats, sample rates and if the card has hardware volume control.**

Now let's do a quick test to see if we're right.

_This will be loud so take your headphones off and/or turn down whatever you Pi is connected to!!!_

While no other audio is being played:

_*speaker-test does not support S24_LE for some reason?_
```
speaker-test -l1 -c2 -Dhw:<card #>,<device #> -r<samplerate> -F<format>
```
So for example if we wanted to make sure the card used in the above examples supported CD quality audio:
```
speaker-test -l1 -c2 -Dhw:0,0 -r44100 -FS16_LE
```

If you don't get any errors and you hear white noise you're all set.

You can use all of your new found information to configure a functional `default` output with the help of the below `asound.conf` by changing:

```
defaults.ctl.card
defaults.pcm.card
defaults.pcm.device
```
`slave.pcm`

in 

`pcm.!default`

And optionally installing higher quality sample rate converters and uncommenting `defaults.pcm.rate_converter`.

Edit to your needs and copy and paste to `/etc/asound.conf` (the file does not exist by default).  

```
  # /etc/asound.conf

###############################################################################

pcm.hqstereo20 {
    @args [
        SAMPLE_RATE FORMAT BUFFER_PERIODS
        BUFFER_PERIOD_TIME VOL_MIN_DB
        VOL_MAX_DB VOL_RESOLUTION VOL_NAME
    ]
    # Sampling rate in Hz:
    # 44100, 48000, 88200, 96000...
    # Defaults to 44.1 kHz (CD Quality).
    @args.SAMPLE_RATE {
        type integer
        default 44100
    }
    # Format:
    # S16_LE, S24_LE, S24_3LE, S32_LE...
    # Defaults to S16_LE (CD Quality).
    @args.FORMAT {
        type string
        default S16_LE
    }
    # Periods per buffer.
    @args.BUFFER_PERIODS {
        type integer
        default 4
    }
    # Period size in time.
    # Defaults to 125ms (0.125 sec).
    # BUFFER_PERIODS * BUFFER_PERIOD_TIME = buffer time/size
    @args.BUFFER_PERIOD_TIME {
        type integer
        default 125000
    }
    # Minimal dB value of the software volume control.
    @args.VOL_MIN_DB {
        type real
        default -51.0
    }
    # Maximal dB value of the software volume control.
    @args.VOL_MAX_DB {
        type real
        default 0.0
    }
    # How many steps between min and max volume.
    @args.VOL_RESOLUTION {
        type integer
        default 256
    }
    # The name of the software volume control.
    # If your card does not have hardware volume control
    # naming it PCM will cause most apps that use alsa volume
    # to use the software volume control.
    @args.VOL_NAME {
        type string
        default Softvol
    }
    type softvol
    min_dB $VOL_MIN_DB
    max_dB $VOL_MAX_DB
    resolution $VOL_RESOLUTION
    control {
        name $VOL_NAME
        card {
            @func refer
            name defaults.ctl.card
        }
    }
    slave.pcm {
        type plug
        slave.pcm {
            type dmix
            ipc_key {
                @func refer
                name defaults.pcm.ipc_key
            }
            ipc_gid {
                @func refer
                name defaults.pcm.ipc_gid
            }
            ipc_perm {
                @func refer
                name defaults.pcm.ipc_perm
            }
            slowptr 1
            hw_ptr_alignment roundup
            slave {
                pcm {
                    type hw
                    nonblock {
                        @func refer
                        name defaults.pcm.nonblock
                    }
                    card {
                        @func refer
                        name defaults.pcm.card
                    }
                    device {
                        @func refer
                        name defaults.pcm.device
                    }
                    subdevice {
                        @func refer
                        name defaults.pcm.subdevice
                    }
                }
                channels 2
                period_size 0
                buffer_size 0
                buffer_time 0
                period_time $BUFFER_PERIOD_TIME
                periods $BUFFER_PERIODS
                rate $SAMPLE_RATE
                format $FORMAT
            }
            bindings {
                0 0
                1 1
            }
        }
    }
}

###############################################################################

# Change to the card number that you want to be the default control card.
# Default: 0
defaults.ctl.card 0

# Change to the card number that you want to be the default playback card.
# It should usually be the same as defaults.ctl.card.
# Default: 0
defaults.pcm.card 0

# Change to the device number that you want to be the default device on the default card.
# 0 or 1 is usually the correct device number.
# Default: 0
defaults.pcm.device 0

# Change to the subdevice number that you want to be the default subdevice on the default device.
# Should rarely need to be changed.
# Default: -1
defaults.pcm.subdevice -1

# To install high quality samplerate converters on Debian based systems:
# sudo apt install -y --no-install-recommends libasound2-plugins 

# To list available rate_converter's:
# echo "$(ls /usr/lib/*/alsa-lib | grep "libasound_module_rate_")" | sed -e "s/^libasound_module_rate_//" -e "s/.so$//"

# Uncomment and replace speexrate_medium with the rate_converter of your choice.
# defaults.pcm.rate_converter speexrate_medium

pcm.!default {
    type empty
    # Optional args:
    # SAMPLE_RATE: default: 44100 
    # FORMAT: default: S16_LE
    # BUFFER_PERIODS: default: 4
    # BUFFER_PERIOD_TIME: default: 125000
    # VOL_MIN_DB: default: -51.0
    # VOL_MAX_DB: default: 0.0
    # VOL_RESOLUTION: default: 256
    # VOL_NAME: default: Softvol

    # Example:
    # hifiberry dac+ zero on a pi zero.
    # slave.pcm "hqstereo20:FORMAT=S32_LE,BUFFER_PERIOD_TIME=250000,VOL_MIN_DB=-48.0"

    slave.pcm "hqstereo20"
}

###############################################################################

ctl.!default {
    type hw
    card {
        @func refer
        name defaults.ctl.card
    }
}
```

### Credits

By:

[JasonLG1979](https://github.com/JasonLG1979)

Inspired greatly by the [install-bluetooth script](https://github.com/nicokaiser/rpi-audio-receiver/blob/master/install-bluetooth.sh) by [nicokaiser](https://github.com/nicokaiser) and others in the [rpi-audio-receiver](https://github.com/nicokaiser/rpi-audio-receiver) repo.

Bluealsa systemd integration adapted from the [Arkq/bluez-alsa wiki](https://github.com/Arkq/bluez-alsa/wiki/Systemd-integration)
