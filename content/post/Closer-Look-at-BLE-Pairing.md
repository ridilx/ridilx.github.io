---
title: "BLE Series | Part 2: A Closer Look at BLE Pairing"
author: "Rida IDIL"
date: 2019-11-20T12:59:46+02:00
image: /img/BLE/1/0.png
subtitle: "Dive deep into Bluetooth Low Energy"
tags: ["BLE", "IoT"]
---

{{< toc >}}

---


## Overview
In Part 1 of this series, we explored BLE packets from the advertising to connection events. Part 2 examines the process of pairing used to establish a secure connection with BLE and extract the passkey used for Authentification from the flash memory.
There is two mode for connection to provide pairing, LE Legacy connections (supported in Bluetooth 4.0 and forward) and LE Secure connections (supported only in Bluetooth 4.2). Each mode has paring methods: Just works, Passkey entry, Out Of Band and Numeric Comparison (Available just in LE Secure connections). For more details check this previous review Deep Dive Into Bluetooth LE Security.
This post will focus only on the LE Secure Connections and have an inside look at the association model Numeric Comparison.
The process of paring involves 3 phases:


---

## Phase 1 - Pairing Attributes

![Pairing request packet](/img/BLE/2/1.png)

In phase 1, the Central device addresses the slave with a Pairing Request packet to exchange its own paring parameters.
1. **Opcode:** the type of paring packet (Request or Response).
2. **IO Capability:** the central device is a smartphone, Keyboard is the output and Display is the input.
3. **Authentification Requirement:** several fields to specify what can Central devise supply for the authentification process. Secure Connection if the device support LE secure Connections (Bluetooth 4.2). Keypress for notification during a passkey entry. MITM protection from Man-In-The-Middle attacks.
![Pairing response packet](/img/BLE/2/2.png)
The peripheral device replies with his pairing parameters and both Input/output capabilities of each device can define the paring method (in this case Numeric comparison).
4. **Initiator/Responder Key Distribution:** as we can see the **LTK** (Long-term key) is chosen as a generated key because the *LE Secure connections* are used instead of **Link Key** (with flag 0) used in **LE Legacy Connection**. *Connection Signature Resolving Key* (**CSRK**) for data signing and *Identity Resolving Key* (**IRK**) used to generate a private MAC address.



---

## Phase 2 - Pairing Over SMP

After exchanging paring features, phase two is for *Long Term Key* (**LTK**) generation. The strength of *LE Secure connections* is the Elliptic Curve Diffie-Hellman (**ECDH**) algorithm used to distribute the secret keys between the two devices which can protect against passive and active attacks.

* **First step:** generate ECDH public and private key in each device and the public keys are shared (Note that the public key contains two parts X and Y each one is 32 bytes long):

![](/img/BLE/2/3.png)
![](/img/BLE/2/4.png)

* **Second Step:** each device selects a random 128-bit nonce, and compute a confirmation value using the public ECDH keys and the random nonces, then the random 128-bit nonce used for calculation is exchanged between the two devices.

![Confirmation value sent from responding device to the initiation device.](/img/BLE/2/5.png)
![Random nonce](/img/BLE/2/6.png)

* **Third Step:** both devices calculate a 6-digit value (from ``000000`` to ``999999``) to be displayed on each devices side.
The example of numeric comparaison in Central device:

![](/img/BLE/2/7.png)

* **Fourth Step:** assuming that the 6-digit authentification succeeds « the end of Authentification stage 1 », the devices start generating the Long Term Key (LTK), and each device sends a DHKey checking packet « the Authentification stage 2 ends ».

![](/img/BLE/2/8.png)
![](/img/BLE/2/9.png)



---

## Phase 3 - Key Distribution

During this phase, the LTK generated will be used to encrypt the connection link, and the specific keys for encryption can be distributed between the devices.

![](/img/BLE/2/10.png)
![](/img/BLE/2/11.png)

With version 4.2 of Bluetooth, BLE offers strong security characteristics to ensure authentification, prevent against passive eavesdropping and MITM attacks with the implementation of the Elliptic Curve Diffie-Hellman algorithm and the Numeric Comparison association method.


---

## Proof of Concept Attack
The LE Secure Connections pairing method encrypts communications between the devices with the Elliptic Curve Diffie-Hellman algorithm. The idea is to dump the RAM memory, extract a copy of the firmware and lock for interesting pieces of information. The complexity of this process depends on the design of the target hardware and the implementation of the firmware.
We are going to develop a peripheral BLE firmware, with fast advertising and LE Secure Connection pairing using Numeric comparison in ``nRF52840`` Development Kit.

![](/img/BLE/2/12.png)

The passkey is ``137343``, the same 6-digit in the screenshot from the Central device ( Phase three ).

![The result of the memory dump](/img/BLE/2/13.png)


The result of the memory dump appears in hexadecimal format, the values in the red frame are the virtual addresses.
The count of the 6-digit is done in flash memory, so there is a higher possibility that he still located in memory.
After parsing the dump, the value shared between the two devices for authentication can be found in plain text:

![](/img/BLE/2/14.png)

And we can extract more sensitive informations like cipher keys.


---

## Conclusion
The security of IoT devices requires both hardware and firmware built-in security. The *Secure Element* (**SE**) approach can ensure a higher level of hardware security, providing an independent chip contains private keys and secrets.
It is true that the Secure Element can protect keys from hardware attacks, but the communication between the MCU and the SE remains exposed, and a secure communication scheme between the two chips is essential.
A robust security design should include the combination of right choices of chip components, the Secure Element, and a correct firmware implementation. This fulfills the requirements of the defense-in-depth concept.