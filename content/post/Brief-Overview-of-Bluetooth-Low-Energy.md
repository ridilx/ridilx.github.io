---
title: "BLE Series | Part 1: A Brief Overview of Bluetooth Low Energy"
author: "Rida IDIL"
date: 2019-10-23T12:59:46+02:00
image: /img/BLE/1/0.png
subtitle: "Dive deep into Bluetooth Low Energy"
tags: ["BLE", "IoT"]
---

{{< toc >}}

---

## Introduction
Bluetooth Low Energy (BLE) is a wireless technology used by several projects applications: Internet of Things (IoT), health care (measuring heart rate ), smart cities and domotic home technologies. For IoT devices, BLE presents many advantages in performance metrics such as ensuring very low power consumption, reliability, BLE can support communication with different mobile devices iOS, android, blackberry, as well as Linux, Mac OS, and Windows operating systems.
In this post, we will describe different packets upcoming from peripheral and central devices to understand how BLE Protocol works.


---

## Bluetooth Low Energy Advertising Packets
On one side, when a BLE peripheral device is turned on, it will periodically send Broadcast packets on specific advertising channels 37, 38, 39 (2402MHz, 2426MHz, 2480MHz respectively) (Bluetooth Low Energy has 40 channels in the ISM band «2.4GHz» spaced with 2MHz frequency for communication) to indicate to other central devices that it is up and ready to receive connections, and on the other side, central devices are listening into those specific channels for upcoming future connection and need only to scan those 3 channels.
During the advertising phase, there is a time interval between each packet sent. We call it the BLE Advertising Interval. He is one of the numerical values used to configure the advertising parameters :

```go {linenos=table,linenostart=0}
adv_params.type        = BLE_GAP_ADV_TYPE_ADV_IND;
adv_params.p_peer_addr = NULL;
adv_params.interval    = 64;
adv_params.timeout     = 180;
```

BLE technology is designed following the concept of layers, Bluetooth Low Energy Link Layer is in charge of sending advertising packets and establishing connections between devices. BLE LL has 2 type of PDU in only one format packet, advertising channel packets, and data channel packets. The choice of the type of packets depends on the channels used for transmission.
1. **Advertising channel:** for advertising and discovering central devices without prior connection state. Advertising uses channels 37, 38 and 39.
2. **Data channel:** after establishing the connection Data Channel packets are transmitted between the two devices. Data channel uses channels from 0 to 36 included.

![](/img/BLE/1/1.png)

The figure above shows a capture with Wireshark of an advertising packet from a peripheral device.
The packet has the following attributes :
1. **Access Address:** 4 bytes length, configured with the default hex value 0x8E89BED6 and used to identify the radio communication on the physical link.
2. **Packet (or advertising) Header** has the following attributes :
* **PDU type:** to describe the advertising channel type. ADV_IND means the device is connectable with undirected advertising. The PDU type ADV_IND give the possibility to the peripheral device to be in mode discoverable when he wants another unspecific master device to initiate a connection.
* **RFU:** reserved for future uses.
* **TxAddr:** it can take 4 values depending on the type of the MAC address configured in the GAP parameters :
```go {linenos=table,linenostart=0}
ble_gap_addr_t gap_addr;
gap_addr.addr_type = BLE_GAP_ADDR_TYPE_PUBLIC; //Public address 0x00
gap_addr.addr_type = BLE_GAP_ADDR_TYPE_RANDOM_STATIC; //Random static address 0x01
gap_addr.addr_type = BLE_GAP_ADDR_TYPE_RANDOM_PRIVATE_RESOLVABLE; //Random private resolvable address 0x02
gap_addr.addr_type = BLE_GAP_ADDR_TYPE_RANDOM_PRIVATE_NON_RESOLVABLE; //Random private non-resolvable address 0x03
```
3. **Adv. Address:** MAC address fixed to 6 bytes length of the device ( we can change the Mac address with a random one )
4. **Adv. Data:** for GAP configuration :
* **Appearance:** defines the external appearance of the device. Like GENERIC_PHONE, GENERIC_WATCH or Unknown or unspecified appearance type.
```go {linenos=table,linenostart=0}
ble_advertising_init_t init;
init.advdata.include_appearance      = true;
```
* **Flags:** choose between Bluetooth radio chipsets mode, « BR/EDR » mode for Basic Rate/Enhanced Data Rate and/or « LE » mode for Low Energy.
```go {linenos=table,linenostart=0}
init.advdata.flags    = BLE_GAP_ADV_FLAGS_LE_ONLY_GENERAL_DISC_MODE; //Which means the peripheral is set up to LE General Discoverable Mode (bit 1) and BR/EDR not supported (bit 0).
```
* **128-bit Service Class UUIDs:** the peripheral devices known as GATT server holds objects called Services that contain specific data called Characteristics. Each service has a unique numeric identifier called UUID to distinguish between Services which can be 16-bit (2 bytes) or 128-bit (16 bytes) length.
```go {linenos=table,linenostart=0}
ble_uuid_t adv_uuids[] = {LBS_UUID_SERVICE, m_lbs.uuid_type};
init.advdata.uuids_complete.p_uuids  = adv_uuids;
```

---

## Bluetooth Low Energy Scan Request and Scan Response Packets

![](/img/BLE/1/2.png)
![](/img/BLE/1/3.png)

After broadcasting useful data during the advertising event, the central devices (Phones and tablets) keep request additional data from the peripheral device with the Scan Request packet, then the peripheral then replies with the Scan Response Packet.


## Bluetooth Low Energy Connect Request Packet
The Central device, « **the initiator** », sends the packet Connect_Req to a device with the connectable and discoverable mode to establish a connection link. This packet contains all the required data needed for the future connection between the two devices.

![](/img/BLE/1/4.png)

The data transmitted in the Link Layer Data represents the new connection setup :
1. **Access Address:** new 4 bytes value different from the default Access Address mentioned in the advertising phase. This new address has been generated randomly by the central device, not a fixed address like in advertising PDU.
2. **CRC Init:** initialization value of CRC.
3. **Window size and offset:** the window size and offset fields represent the timing for the first data packet and the transmit window start respectively.
4. **Latency:** define the Slave « **peripheral** device » latency, to save power on the device. We can define the number of time the slave can skip some connection events when there is no data.
5. **Channel Map:** which RF channels will be used and which will not be used. « ``0xff ff ff ff 1f`` » means we can use channels from 1 to 36 dedicated to Data.
6. **Hop:** the Random number used as a parameter in an algorithm to calculate unmapped channels of the current connection event.

Note that these initial connection parameters are set up by the central device, but the peripheral can do an update to the connection parameters to change them :

```go {linenos=table,linenostart=0}
ble_gap_conn_params_t   gap_conn_params;
gap_conn_params.min_conn_interval = MIN_CONN_INTERVAL;
gap_conn_params.max_conn_interval = MAX_CONN_INTERVAL;
gap_conn_params.slave_latency     = SLAVE_LATENCY; //0x00
gap_conn_params.conn_sup_timeout  = CONN_SUP_TIMEOUT;
sd_ble_gap_ppcp_set(&gap_conn_params);
```
<div class="box-error">
From here, we gonna switch from advertising channel to data channel PDU. The same format packet but the structure of PDU is different.
</div>

## Bluetooth Low Energy Data Channel Packet
We can recognize 3 type of Data Channel Packet by *LLID* ( **Link Layer ID** ) one of the components of the PDU Header :

![LLID 1](/img/BLE/1/5.png)

![LLID 2](/img/BLE/1/6.png)



Note that the new Access Address ( ``0xbc75ca22`` ) used is the one generated by the central device.
* **Type 1 « LLID == 0x01»:** called Empty PDU and also known as acknowledges packet, if the peripheral device has « ``0x1`` » ( our case ) as value, he will reply to every connection event packet sent from the Central device at every connection interval. The type LLID ``0x1`` may signify whether it is an empty PDU or it is a continuation fragment of L2CAP packets.

* **Type 2 « LLID == 0x02»:** called Data PDU, In addition to Bluetooth Low Energy Link Layer, there are two other higher protocol Layers added, L2CAP and Security Manager Protocol. The L2CAP plays the role of support of a higher level protocol multiplexing over the same physical link. With L2CAP we can assign to SMP ( Security Manager protocol ) and ATT ( Attribute protocol ) a fixed logical channel CID -``0x0006`` for SMP and ``0x0004`` for ATT -.

* **Type 3 « LLID == 0x03 »:** called LL Control PDU to control the link layer connection. The payload of the PDU consists of:
1. **Opcode:** 1-byte length, **LL_FEATURE_REQ** for features set up by the master's Link Layer.
2. **Control data:** feature Set is specified by the opcode **LL_FEATURE_REQ**, in our example, the LE Encryption will be established if the LL_FEATURE_RSP from the peripheral will agree.

![](/img/BLE/1/8.png)


---

## Security Manager Protocol

The **SMP** resides on top of L2CAP to implement the security features between devices, by setting the secure connection flag, the MITM ( Man-In-The-Middle) flag, the Bonding flag, generate encryption keys, identity keys and handle the key distribution process in the pairing mode.

```go {linenos=table,linenostart=0}
ble_gap_sec_params_t sec_param;
//Security parameters to be used for all security procedures
sec_param.bond       = 1;
sec_param.mitm       = 1;
sec_param.lesc       = 1;
sec_param.keypress   = 1;
```

---

## Attribute Protocol

The ATT protocol is based on client server communication, each device has its data organized in the form of attributes.
The client sends the Read By Type Request to obtain the attribute handler between the Starting Handle ``0x0001`` and the Ending Handle ``0X009``:

![Attribute Request packet](/img/BLE/1/9.png)

The server replies with Read By Type Response with contains the attribute data handles, characteristic handles and UUID's:

![Attribute Response packet](/img/BLE/1/10.png)

---

## Conclusion
The following figure emphasizes the role of each BLE device, from the process of Advertising to establishing the connection.

![conclusion](/img/BLE/1/11.png)
