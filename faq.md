---
layout: page
title:  "FAQ"
date:   2019-07-26 13:18:23 +0100
permalink: /faq/
---

## What is the TPM?
The Trusted Platform Module (TPM) is an  is an international standard for a secure cryptoprocessor...

## Why should I use a TPM?
Quote NIST

## How do I get a TPM

-> check if you have a tpm (tpm.msc, /dev/tpm0, /dev/tpmrm0)
-> TPM for Raspberry Pi (iridum, ms, letstrust)
-> tpm simulator(s)

## What is the TSS?

The TPM Software Stack (TSS) is a library to facilitate communication with the TPM.

## How to install the TSS on my system?
**Ubuntu**
  - xenial:
    ```bash
    sudo apt install libtss2-0 libtss2-dev libtss2-utils
    ```
  - bionic, cosmic:
    ```bash
    sudo apt install libsapi0 libsapi-dev libsapi-util
    ```
  - disco, eoan:
    ```bash
    sudo apt install libtss2-dev libtss2-esys0 libtss2-udev
    ```

**Fedora**
```bash
sudo yum install tpm2-tss-devel
```
**Arch**
```bash
sudo pacman -Syu tpm2-tss
```
**openSuse**
```bash
sudo zypper in libtss2-esys0
```

## How to install the TSS from source
{% highlight bash %}
git clone https://github.com/tpm2-software/tpm2-tss
cd tpm2-tss
./bootstrap
./configure --enable-integration
make -j$(nproc)
make check
{% endhighlight %}

Check out the [tpm2-tss README] and [tpm2-tss INSTALL] for more information.

[tpm2-tss README]: https://github.com/tpm2-software/tpm2-tss/blob/master/README.md
[tpm2-tss INSTALL]: https://github.com/tpm2-software/tpm2-tss/blob/master/INSTALL.md

## Tutorial: tpm2-tools



## What is the difference between a TPM and a HSM?