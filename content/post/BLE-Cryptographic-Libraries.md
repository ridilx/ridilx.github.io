---
title: "BLE Series | Part 3: A Quick Comparison Between Cryptographic Library Backends"
author: "Rida IDIL"
date: 2019-12-04T12:59:46+02:00
image: /img/BLE/1/0.png
subtitle: "Dive deep into Bluetooth Low Energy"
tags: ["BLE", "IoT"]
---

{{< toc >}}

---

## TL;DR
In the previous post «A Closer Look at BLE Pairing», we explored the different phases of pairing with the LE Secure Connection pairing mode. 
Readers are reminded that LE Secure Connection is the other option to set the pairing mode for BLE protocol, it was introduced in version ``4.2`` of Bluetooth to enhance the security features.
The most significant updates to the ``4.1`` version are the use of Elliptic Curve Diffie-Hellman algorithm, one of the FIPS-approved algorithm, and the support of a new association model named Numeric Comparison.
This post will focus on each cryptography library backends and the common API provided. Moreover, we will test the effectiveness of each one to enhance the ability to make the right implementation for BLE IoT projects.


---

## Test Environment Setup

All developments have been conducted on an nRF52 Nordic development kit platform:

* The currently supported Nordic board is: ``nRF52840``
* The currently supported hardware chip: **PCA10056 ARM Cortex-M4F CPU**
* The currently supported SDK version is: ``14.2.0``
* The currently supported Softdevice version is: ``s140_5.0.0``

## The Implementation of the Cryptographic Library

### Where to start?

Enable the cryptography library module into the header file ``"sdk_config.h"``:

```go {linenos=table,linenostart=0}
//==========================================================
// <e> NRF_CRYPTO_ENABLED - nrf_crypto - Cryptography library
//==========================================================
#ifndef NRF_CRYPTO_ENABLED
#define NRF_CRYPTO_ENABLED 1
#endif
```

The Cryptographic Library offers two types of crypto context usage :

1. **Encryption with ECDH (Elliptic Curve Diffie-Hellmann):** it can be considered a key agreement protocol, more than an encryption algorithm. ECDH basically manages how Public & Private keys should be generated and establishes a shared secret (DHKEY) over an unencrypted communication link to be exchanged between the communication end point. (needed for Bluetooth LE Secure Connections pairing mode)

2. **Signing with ECDSA (Elliptic Curve Digital Signature Algorithm):** a variant of Digital Signature Algorithm. It is needed to sign firmware uploading with Secure DFU mode.

The common API offers two backends to implement LE Secure Connections with Elliptic Curve Cryptography:

![](/img/BLE/3/1.png)

* **CC310_LIB** (CryptoCell) :
<div class="box-warning">
The hardware-accelerated backend uses cryptography hardware to perform cryptographic operations. The required CryptoCell hardware is available on the nRF52840 SoC, but not on the nRF52832 SoC. See the nRF52840 Product Specification for detailed information about CryptoCell.
</div>

* **MICRO_ECC** :

<div class="box-warning">
micro-ecc is an open source library that performs cryptographic operations. It is optimized for small program size while maintaining high processing speed. The micro-ecc backend supports all current nRF5 Series devices.
</div>

Select the best backend depends on the project use cases and resource criteria, for that we need to enable some module from the configuration header file :

#### CC310 Case:

```go {linenos=table,linenostart=0}
// <q> NRF_CRYPTO_BACKEND_CC310_LIB  - Enable the ARM Cryptocell CC310 backend


// <i> The hardware-accelerated cryptography backend is available only on nRF52840.
#ifndef NRF_CRYPTO_BACKEND_CC310_LIB
#define NRF_CRYPTO_BACKEND_CC310_LIB 1
#endif
```

#### MICRO-ECC Case:

```go {linenos=table,linenostart=0}
// <e> NRF_CRYPTO_BACKEND_MICRO_ECC - Enable the micro-ecc software backendRNG_CONFIG_POOL_SIZE
// <i> The micro-ecc library provides a software implementation of ECC cryptography for nRF5 Series devices.
//==========================================================
#ifndef NRF_CRYPTO_BACKEND_MICRO_ECC
#define NRF_CRYPTO_BACKEND_MICRO_ECC 1
#endif
// <q> NRF_CRYPTO_BACKEND_MICRO_ECC_SHA256  - Enable SHA256


// <i> Enable SHA256 cryptographic hash functionality.
// <i> Enable this setting if you need SHA256 support, for example to verify signatures.
#ifndef NRF_CRYPTO_BACKEND_MICRO_ECC_SHA256
#define NRF_CRYPTO_BACKEND_MICRO_ECC_SHA256 1
#endif
// <q> NRF_CRYPTO_BACKEND_MICRO_ECC_RNG  - Enable random number generator


// <i> Enable random number generation.
// <i> Enable this setting if you need to generate cryptographic keys.
// <i> This setting requires the RNG peripheral driver to be present.
#ifndef NRF_CRYPTO_BACKEND_MICRO_ECC_RNG
#define NRF_CRYPTO_BACKEND_MICRO_ECC_RNG 1
#endif
```

### Source files enumeration

![Sources and header files for mirco-ecc backend](/img/BLE/3/2.png)

The « ``micro-ecc`` » library sources files can handle :
* The initialization of the backend
* The generator of public & private keys and compute the **shared secret**
* **ecdh** & **ecdsa** implementation


![Sources and headers files for cc310 backend](/img/BLE/3/3.png)

The « cc310 » library sources files can handle :
* The initialization of the backend
* The generator of public & private keys and compute the **shared secret**
* **ecdh** & **ecdsa** implementation
* hash and generator of random numbers functions

### What about a different version of SDK?

<div class="box-error">
**Be warned:** each version of SDK may have different modules, in other words, the name and the functionality of modules in the SDK configuration header file « sdk_config.h » can be changed from version SDK version to another.
Error
</div>
---

## CC310 vs MICRO-ECC

### What does official Nordic documentation says about backends?

<div class="box-note">
The hardware-accelerated backend is faster, consumes less power, and offers more functionality. Using the hardware-accelerated backend also decreases the size of the application. However, this backend requires hardware support and is therefore available only on the nRF52840 SoC.
</div>

### Key comparisons

In order to compare the two backends, we will implement a Peripheral App. firmware that support Passkey as pairing method for *LE Secure Connections* (4.2 devices only); LE Security Mode: ``1`` and Level : ``4``;

### What do I think about execution time performance?

Code profiling and measure time execution of each cryptography library backend need to enable the module RTC (Real-Time Counter). 
The module RTC uses a low-frequency source, this involves that the firmware will consume less power and less resolution.

```go {linenos=table,linenostart=0}
// <e> RTC_ENABLED - nrf_drv_rtc - RTC peripheral driver
//==========================================================
#ifndef RTC_ENABLED
#define RTC_ENABLED 1
#endif
...
// <q> RTC2_ENABLED  - Enable RTC2 instance
#ifndef RTC2_ENABLED
#define RTC2_ENABLED 1
#endif
```

The point here is to measure the elapsed time to complete the tasks listed below :
* Task for initializing the cryptography library
* Task to generate of the public and private key pair
* Task to set the public key into the peer manager handler

![](/img/BLE/3/4.png)


#### MICRO-ECC

![](/img/BLE/3/5.png)

![](/img/BLE/3/6.png)

After multiple simultaneous test, both values indicates :
``0.113 second`` for setting up the LE Secure connections parameters
``0.110 second`` to compute the shared


#### CC310

![](/img/BLE/3/7.png)

![](/img/BLE/3/8.png)

After multiple simultaneous test, both values indicates :
``0.43 second`` for setting up the LE Secure connections parameters
``0.14 second`` to compute the shared

<div class="box-note">
Contrary to what was said in the statement of Nordic documentation, the software backend « micro-ecc » is more faster than the hardware-accelerated backend « cc310 ».
</div>

### What do I think about power consumption?
The power consumption is an important parameter that influence the average of the battery life.
To be able to estimate the difference of power consumption between the two cryptography library backends, we will measure the average current draw over time by the ``nRF52840`` board. As the **power** is just the result of the multiplication operation between the **voltage** and the **current**. With fixed value of **voltage**, the variation of the **current** consumption can reflect proportionally the **power consumption**.


#### ``nRF52840`` current measurement with « micro-ecc »

<iframe width="100%" height="500px" src="https://www.youtube.com/embed/Vy2wnSuquzE?vq=hd1080" allowfullscreen>
</iframe>

Using an ampere meter to measure the average current during the advertising and connection events, the values obtained vary between ``0.6mA`` and ``1.7mA``


#### ``nRF52840`` current measurement with « cc310 »

<iframe width="100%" height="500px" src="https://www.youtube.com/embed/zkvcldw_Vnw?vq=hd1080" allowfullscreen>
</iframe>


Using an ampere meter to measure the average current during the advertising and connection events, the values obtained vary between ``6.8mA``and ``7.7mA``.

Contrary to what was said in the statement of Nordic documentation, the software backend « **micro-ecc** » consumes less power than the hardware-accelerated backend « **cc310** ».


### What do I think about portability and functionality?

On the left firmware with CC310, on the right firmware with MICRO-ECC

![](/img/BLE/3/9.png)
![](/img/BLE/3/10.png)

Firmware file size with Micro-ecc: ``2 146 428 bytes``
Firmware file size with CC310: ``2 205 020 bytes``

The difference between the two values is ``58 592 bytes``

And contrary to what was said in the statement of Nordic documentation, the software backend « **micro-ecc** » decrease the size of the application (firmware) compared to the hardware-accelerated backend « **cc310** ».
Why does it matter ?
With real IoT product, the portability, the functionality, the speed and the accuracy of the IoT object and battery life are the factors that matter for a better user experience. However, security implementation can affect performance regarding these factors.