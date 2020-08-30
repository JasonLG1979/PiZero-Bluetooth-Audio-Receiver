# PiZero-Bluetooth-Audio-Receiver
## How to turn your Raspberry Pi Zero W into a bluetooth audio receiver.

The goal of this howto is to turn a Raspberry Pi Zero W into a headless bluetooth audio receiver utilizing it's onboard bluetooth module, with packages available in the default repositories, starting from a fresh Raspberry Pi OS Lite install.

### Expectations
It is assumed that you have started from a fully updated, unmodified, and fresh Raspberry Pi OS Lite install, have shell access, and at least very basic Linux knowledge.

### Limitations
The Raspberry Pi Zero is not a very powerful device and the onboard bluetooth module is not the greatest, it is known to have wifi coexistence issues (you may experience audio dropouts with wifi enabled), it is not suitable for low latency aplications and the package used in this howto (bluealsa) only supports the SBC codec (no AAC, aptX*, LDAC).

<i>*The dropout issue can sometimes be mitigated by disabling wifi and/or forcing turbo mode (the CPU will run full tilt all the time). Forcing turbo will however cause your Pi Zero to use more power and produce more heat. Under normal circumstances though heat is not a concern with a Pi Zero even with force turbo enabled.</i>
