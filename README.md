# PiZero-Bluetooth-Audio-Receiver
## How to turn your Raspberry Pi Zero W into a bluetooth audio receiver.

The goal of this howto is to turn a Raspberry Pi Zero W into a headless bluetooth audio receiver utilizing it's onboard bluetooth module, with packages available in the default repositories, starting from a fresh Raspberry Pi OS Lite install.

### Expectations
It is assumed that you have started from a fully updated, unmodified, and fresh Raspberry Pi OS Lite install, have shell access, at least very basic Linux knowledge and have some way of getting audio out of the Pi Zero. (the Pi Zero has no analog audio output) 

### Limitations
The Raspberry Pi Zero is not a very powerful device and the onboard bluetooth module is not the greatest, it is known to have wifi coexistence issues (you may experience audio dropouts with wifi enabled), it is not suitable for low latency aplications and the package used in this howto (bluealsa) only supports the SBC codec (no AAC, aptX*, or LDAC). If you're looking for an HD audiophile experience, this ain't it.

<i>*The dropout issue can sometimes be mitigated by disabling wifi and/or forcing turbo mode (the CPU will run full tilt all the time). Forcing turbo will however cause your Pi Zero to use more power and produce more heat. Under normal circumstances though heat is not a concern with a Pi Zero even with force turbo enabled.</i>

### On with it then...

Install the packages needed:

 ```sudo apt install -y --no-install-recommends alsa-base alsa-utils bluealsa bluez-tools```


Create a bluealsa group:

```sudo addgroup --system bluealsa```


Create an unprivileged bluealsa system user in the bluealsa group:

```sudo adduser --system --disabled-password --disabled-login --no-create-home --ingroup bluealsa bluealsa```


Add the bluealsa user to the bluetooth group:

```sudo adduser bluealsa bluetooth```


Add the bluealsa user to the audio group:

```sudo adduser bluealsa audio```


Create an unprivileged bt-agent system user in the bluetooth group:

```sudo adduser --system --disabled-password --disabled-login --no-create-home --ingroup bluetooth bt-agent```


Edit ```/etc/bluetooth/main.conf``` to disable the discoverable timeout and change our device class to "HiFi Audio Device":

Change ```#Class = 0x000100``` to ```Class = 0x200428``` and ```#DiscoverableTimeout = 0``` to ```DiscoverableTimeout = 0```.

Save and exit nano (ctrl+x, y, enter)


Create an override for the bluetooth.service that disables unneeded plugins:

```sudo systemctl edit bluetooth.service```

Paste this into the file:
```
[Service]
ExecStart=
ExecStart=/usr/lib/bluetooth/bluetoothd  --noplugin=sap,network,hog,health,midi
```
Save and exit nano (ctrl+x, y, enter)


Edit ```/etc/dbus-1/system.d/bluealsa.conf``` to allow our unprivileged bluealsa system user to run the bluealsa daemon.

```sudo nano /etc/dbus-1/system.d/bluealsa.conf```

Change ```<policy user="root">``` to ```<policy user="bluealsa">```

Save and exit nano (ctrl+x, y, enter)


Override the bluealsa.service file:
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


Reload the systemd daemon:

```sudo systemctl daemon-reload```


Create the ```bluealsa-aplay.service``` unit file:

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


Enable the bluealsa-aplay.service:

```sudo systemctl enable bluealsa-aplay.service```


Create the bt-agent service to enable "Just Works" bluetooth pairing:

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


Enable the bt-agent.service:

```sudo systemctl enable bt-agent.service```


Create the bt-discovery service to enabe discoverability at startup and be able to toggle it on and off with systemd:

```sudo nano /etc/systemd/system/bt-discovery.service```

Paste this into the file:
```
[Unit]
Description=Toggle bluetooth discoverable
Requires=bt-agent.service
After=bt-agent.service

[Service]
Type=oneshot
ExecStart=/usr/bin/bluetoothctl discoverable on
ExecStop=/usr/bin/bluetoothctl discoverable off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable the bt-discovery.service:

```sudo systemctl enable bt-discovery.service```


Nuke the useless ```bthelper@.service```:

```sudo systemctl disable bthelper@.service```

```sudo systemctl mask bthelper@.service```

```sudo rm /lib/systemd/system/bthelper@.service```

```sudo systemctl daemon-reload```

```sudo systemctl reset-failed```

Create a udev script so our Pi Zero is only discoverable if no devices are connected:

```sudo nano /usr/local/bin/bluetooth-udev```

Paste this into the file:
```
#!/bin/bash
if [[ ! $NAME =~ ^\"([0-9A-F]{2}[:-]){5}([0-9A-F]{2})\"$ ]]; then exit 0; fi
action=$(expr "$ACTION" : "\([a-zA-Z]\+\).*")
if [ "$action" = "add" ]; then
    systemctl stop bt-discovery.service
fi
if [ "$action" = "remove" ]; then
    deviceinfo=$(bluetoothctl info)
    if [ "$deviceinfo" = "Missing device address argument" ]; then
        systemctl start bt-discovery.service
    fi
fi
```
Save and exit nano (ctrl+x, y, enter)


Make set the proper permissions for the script:

```sudo chmod 755 /usr/local/bin/bluetooth-udev```


Create a udev rule to call the scrpit:

```sudo nano /etc/udev/rules.d/99-bluetooth-udev.rules```


Paste this into the file:
```
SUBSYSTEM=="input", GROUP="input", MODE="0660"
KERNEL=="input[0-9]*", RUN+="/usr/local/bin/bluetooth-udev"
```
Save and exit nano (ctrl+x, y, enter)

Reboot:

```sudo reboot```

Your Pi Zero should be discoverable and show up to other devices as a bluetooth audio receiver. It's name will be whatever your Pi Zero's hostname is.
