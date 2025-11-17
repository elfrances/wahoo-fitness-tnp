# Introduction

This document is an unofficial specification of the wahoo-fitness-tnp service and associated WFTNP protocol. The information herein was obtained by reverse engineering the operation of the protocol, without any reference to the official (proprietary) specification from Wahoo Fitness. 

Wahoo Fitness first intoduced WFTNP in January 2021, when it released its [Direct Connect (DIRCON)](https://www.wahoofitness.com/devices/indoor-cycling/accessories/kickr-dircon-buy) adapter.  The adapter connects to the RJ-11 port on the KICKR V5 indoor trainer, which was Wahoo's flagship trainer at the time, and to the wired Ethernet network via an RJ-45 port.  This adapter has no support for WiFi.

For an in-depth review of this DIRCON dongle, you can read [this](https://www.dcrainmaker.com/2021/01/wahoo-starts-shipping-kickr-2020-direct-connect-cable-hands-on-details.html) blog post by DC Rainmaker or watch [this](https://youtu.be/XtIM5675dLo?si=B5nM_biNNlvfmuu2) Youtube video from Shane Miller.

At the time of this writing (late 2025) WFTNP is available on a wide range of indoor trainers from different manufacturers.  The table below lists some of them:

| Brand | Model | Enet | WiFi |
|-------|-------|:----:|:----:|
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

**Note:** in the rest of this document we use DIRCON and WFTNP interchangeably, although DIRCON is a marketing name for a dongle that uses WFTNP.

# Protocol Overview

At a high level, DIRCON is simply "BLE over TCP/IP".  That is, the Bluetooth Low Energy (BLE) messages normally exchanged between the virtual training app (the BLE client) and the smart trainer (the BLE server) are instead encapsulated and transmitted using a TCP connection over wired Ethernet or WiFi. That's it!

## Service Advertisement

DIRCON uses Multicast DNS (mDNS) to advertise the "wahoo-fitness-tnp" (WFTNP) service on the local network, so that a DIRCON-compatible virtual training app (such as FulGaz, Wahoo SYSTM, or Zwift) can find it.

The screenshot below shows the macOS mDNS browser app "Discovery" having discovered three WFTNP devices on the local network. In this case, the "KICKR CB7D" was a Wahoo KICKR V5 bike trainer connected to the home network using the DIRCON dongle.  

![macOS Discovery app](./assets/Discovery.png)

The mDNS query response sent by the trainer contains three relevant records:

### A Record

The A record indicates the IP address of the trainer on the local network.

### SRV Record

The SRV record indicates the TCP port number used by WFTNP, which by default is 36866.

### TXT Record

The TXT record includes three strings:

**serial-number** indicates the Serial Number of the device.

**mac-address** indicates the IEEE MAC address of the Ethernet/WiFi network interface of the device.

**ble-service-uuids** contains a comma-separated list of the 16-bit UUID of the BLE services supported by the device.

In the above screenshot all three indoor bike devices advertised the Fitness Machine Service (FTMS) and the Cycling Power Service (CPS), with BLE UUID's 0x1826 and 0x1818, respectively.

## Connection Establishment

DIRCON follows the client-server model, where the virtual training app is the client and the smart trainer device is the server.

Once the virtual training app discovers the smart trainer, it uses the IP address and port number obtained from the mDNS response to establish the TCP connection to the trainer.  Once established, client and server can exchange DIRCON messages, with the client generally sending a request, and the server answering with a response. The exception being the unsolicited notifications the server (smart trainer) sends to the client during the activity.

To reduce the communication overhead, TCP (by default) tries to buffer as much data as possible before sending it to the peer.  Typically, the internal buffering time is about 200 ms. But DIRCON is a time-sensitive protocol, so to reduce the transaction latency it is suggested that the TCP connection use the TCP_NODELAY option, which disables Nagle's algorithm.

And to allow the client and the server to detect connection drops, it is recommended that the TCP connection use the TCP_KEEPALIVE option as well.

## Message Format

The figure below shows the generic DIRCON message format. It includes a fixed message header followed by an optional, variable-length, data:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |  Message Type |    Seq Num    |   Resp Code   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |          Data Length          |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                               .                               |
     +                               .                               +
12   |                          Optional Data                        |
     +                               .                               +
16   |                               .                               |
     +                               .                               +
     |                               .                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Version:** Indicates the protocol version number. The current version is 1.

**Message Type:** Indicates the type of message sent or received.

| Message Type | Description |
|:-------:|-------|
| 1 | Discover Services |
| 2 | Discover Characteristics |
| 3 | Read Characteristic |
| 4 | Write Characteristic |
| 5 | Enable Characteristic Notifications |
| 6 | Characteristic Notification |

**Seq Num:** Indicates the sequence number of the message, used to match request and response.

**Resp Code:** Indicates the response code. Set to zero in the request messages. The supported response codes are:

| Resp Code | Description |
|:-------:|-------|
| 0 | Success |
| 1 | Invalid Message Type |
| 2 | Generic Error |
| 3 | Service Not Found |
| 4 | Characteristic Not Found |
| 5 | Characteristic Operation Not Supported |
| 6 | Characteristic Write Failed |

**Data Length:** Indicates the length (in bytes) of the Optional Data that follows the fixed message header. The 16-bit value is stored in network byte order.

**Optional Data:** Present in all message types, except for Discover Services.

While the operation of DIRCON is similar to BLE, there are a few notable differences:

* DIRCON message types are a subset of the BLE message types.  For example, in BLE there is the NOTIFICATION and INDICATION message types, both used by the BLE server to send unsolicited data to the client, with the main difference being that the NOTIFICATION message is "best-effort" while the INDICATION message is "reliable".  But because DIRCON runs over TCP (a reliable transport protocol) **all** its messages are "reliable", hence there is no need for the INDICATION message type.
* In DIRCON all messages, except for Characteristic Notification, are a request-response transaction, so there is no need for BLE's WRITE_WITHOUT_RESPONSE message either.
* In DIRCON the UUID's of all services and characteristics are always 128-bit long. The 16-bit BLE UUID's are encapsulated in a 128-bit UUID using the BLE SIG Base UUID 0000**xxxx**-0000-1000-8000-00805F9B34FB, where the four hex digits **xxxx** encode the 16-bit UUID; e.g. the FTMS UUID 0x1826 would be encoded as 00001826-0000-1000-8000-00805F9B34FB.
* BLE treats the 128-bit UUID as an **unsigned integer value**, stored in memory using BLE's "little-endian" byte ordering; e.g. the UUID 00001826-0000-1000-8000-00805F9B34FB is treated as the unsigned integer 0x0000182600001000800000805F9B34FB and stored in memory as the byte sequence FB349B5F800000800010000026180000. In DIRCON the 128-bit UUID's are treated as a **raw 16-byte value**, stored in memory in the same "left-to-right" order used to represent them; e.g. the UUID 00001826-0000-1000-8000-00805F9B34FB is stored in memory as the byte sequence 0000182600001000800000805F9B34FB.

### Discover Services

After connecting to the server, the client sends a Discover Services request message to find out all the services supported by the server. The format of the message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       1       |    Seq Num    |       0       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |               0               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The response message sent by the server includes a list of the 128-bit UUID's of all the services it supports. Notice that, in general, this list may include more UUID's than the ones stated in the ble-service-uuids text record sent in the mDNS advertisement.  The format of the message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       1       |    Seq Num    |   Resp Code   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |            16 * N             |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                        Service UUID #1                        |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
24   |                                                               |
     +                                                               +
28   |                        Service UUID #2                        |
     +                                                               +
32   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
36   |                               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
     |                               .                               |
     +                               .                               +
     |                               .                               |  
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
     |                                                               |
     +                                                               +
     |                        Service UUID #N                        |
     +                                                               +
     |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Discover Characteristics

The client sends a Discover Characteristics request message to find out all the characteristics supported by the specified service. The format of the message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       2       |    Seq Num    |       0       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |              16               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                          Service UUID                         |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The response message sent by the server includes the service UUID and a list of records, one for each characteristic supported by the specified service.  The record includes the 128-bit UUID of the characteristic, and a byte with the bitwise OR of the properties of the given characteristic. The format of the message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       2       |    Seq Num    |   Resp Code   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |         16 + 17 * N           |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                          Service UUID                         |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
24   |                                                               |
     +                                                               +
28   |                      Characteristic UUID #1                   |
     +                                                               +
32   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
36   |                               |  Properties   |               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               +
40   |                                                               |
     +                                                               +
28   |                      Characteristic UUID #2                   |
     +                                                               +
32   |                                                               |
     +                                               +-+-+-+-+-+-+-+-+
36   |                                               |  Properties   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
40   |                               .                               |
     +                               .                               +
     |                               .                               |
     +                               .                               +
     |                               .                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The available properties are:

| Value | Description |
|:-------:|-------|
| 0x01 | READ |
| 0x02 | WRITE |
| 0x04 | NOTIFY |

### Read Characteristic

The client sends a Read Characteristic request message to get the current value of the specified characteristic.  This message is only applicable to characteristics that support the READ property.  The format of the message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       3       |    Seq Num    |       0       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |              16               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                      Characteristic UUID                      |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The response message sent by the server includes the characteristic UUID followed by its current value. The format of the message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       3       |    Seq Num    |   Resp Code   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |            16 + N             |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                      Characteristic UUID                      |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         Value (N bytes)       +
     |                                                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   
```

Note: when the value is a multi-byte integer (e.g. UINT16, UINT24, UINT32, etc) the value is stored in BLE's "little-endian" format, with the least significant byte first and the most significant byte last.

### Write Characteristic

The client sends a Write Characteristic request message to set the value of the specified characteristic.  This message is only applicable to characteristics that support the WRITE property.  The format of the message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       4       |    Seq Num    |       0       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |            16 + N             |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                      Characteristic UUID                      |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         Value (N bytes)       +
     |                                                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   
```

Note: when the value is a multi-byte integer (e.g. UINT16, UINT24, UINT32, etc) the value is stored in BLE's "little-endian" format, with the least significant byte first and the most significant byte last.

The format of the response message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       4       |    Seq Num    |   Resp Code   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |              16               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                       Characteristic UUID                     |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   
```

### Enable Characteristic Notifications

The client sends an Enable Characteristic Notifications request message to enable or disable unsolicited notifications on the specified characteristic. This message is only applicable to characteristics that support the NOTIFY property.  The format of the message is: 

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       5       |    Seq Num    |       0       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |              17               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                       Characteristic UUID                     |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |    Enable     |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   
```

where the Enable byte is a boolean used to enable (1) or disable (0) the notifications.

The format of the response message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       5       |    Seq Num    |   Resp Code   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |              16               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                       Characteristic UUID                     |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+  
```

### Characteristic Notification

The server sends a Characteristric Notification message to inform the client of an event.  The event can be periodic, or it can be triggered by a previous command sent by the client to request the server to perform some operation.  The format of the message is:

```
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 0   |   Version     |       6       |    Seq Num    |       0       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 4   |            16 + N             |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
 8   |                                                               |
     +                                                               +
12   |                      Characteristic UUID                      |
     +                                                               +
16   |                                                               |
     +                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
20   |                               |                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+         Value (N bytes)       +
     |                                                               |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+   
```

Note: when the notification value is a multi-byte integer (e.g. UINT16, UINT24, UINT32, etc) the value is stored in BLE's "little-endian" format, with the least significant byte first and the most significant byte last.
