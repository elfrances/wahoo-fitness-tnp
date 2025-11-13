# Introduction

This document is an unofficial specification of the wahoo-fitness-tnp service used by the Direct Connect (DIRCON) protocol. The information herein was obtained by reverse engineering the operation of the protocol, without any reference to the official (proprietary) specification from Wahoo Fitness. 

Wahoo Fitness first intoduced DIRCON in January 2021, via an optional [dongle](https://www.wahoofitness.com/devices/indoor-cycling/accessories/kickr-dircon-buy) that connected to the RJ-11 port on the KICKR V5 indoor trainer.  This DIRCON dongle only worked with wired Ethernet; i.e. there was no support for WiFi.

For an in-depth review of this DIRCON dongle, you can read [this](https://www.dcrainmaker.com/2021/01/wahoo-starts-shipping-kickr-2020-direct-connect-cable-hands-on-details.html) blog post by DC Rainmaker or watch [this](https://youtu.be/XtIM5675dLo?si=B5nM_biNNlvfmuu2) Youtube video from Shane Miller.

At the time of this writing (late 2025) DIRCON is available on a wide range of indoor trainers from different manufacturers.  The table below lists some of them:

| Brand | Model | Enet | WiFi |
|-------|-------|------|------|
|Elite|Justo|Y|N|
|Elite|Justo 2|Y|Y|
|Elite|Avanti|Y|Y|
|JetBlack|Victory|N|Y|
|Tacx|Neo 3M|Y|Y|
|Wahoo|KICKR V5|Y|N|
|Wahoo|KICKR V6|Y|Y|
|Wahoo|KICKR BIKE V2|Y|Y|
|Wahoo|KICKR MOVE|Y|Y|
|Wahoo|KICKR BIKE SHIFT|Y|Y|
|Wahoo|KICKR CORE V2|N|Y|

The trend in the smart trainer market is moving toward built-in Wi-Fi connectivity because it offers a more stable and faster connection than traditional Bluetooth or ANT+ for competitive virtual racing.

# Protocol Definition

At a high level, DIRCON is simply "BLE over TCP/IP".  That is, the Bluetooth Low Energy (BLE) messages normally exchanged between the virtual cycling app (the BLE client) and the smart trainer (the BLE server) are instead encapsulated and transmitted using a TCP connection over wired Ethernet or WiFi. That's it.

DIRCON uses the mDNS service "wahoo-fitness-tnp" (WFTNP) to advertise itself on the local network, so that a DIRCON-compatible virtual cycling app (such as FulGaz or Zwift) can find it.  The WFNTP mDNS advertisement include the UUID of the BLE services supported by the smart trainer, which typically include FTMS (0x1826) and CPS (0x1818).  The screenshot below shows the macOS mDNS browser app "Discovery" having discovered three DIRCON devices on the local network:




