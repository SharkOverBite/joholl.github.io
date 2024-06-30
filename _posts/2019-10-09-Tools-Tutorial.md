---
layout: post
title:  "TPM Tutorial"
date:   2019-10-09 18:28:05 +0100
categories: tpm2-tools tutorial
tags: bash make code
---


## Introduction

You want to learn to use the TPM 2.0? Then you're right here! In this tutorial
you will get an overview over the TPM using the [tpm2-tools].

The TPM (Trusted Platform Module) is a cryptographic processor which is part of
most modern motherboards. If your computer runs Windows 10, it [certainly
contains a TPM 2.0]. For those unfamiliar, the TPM provides out-of-band general
cryptographic, storage, policy and key management operations (among other
things).

## Getting Started

Of course we need a TPM. If you do not have a TPM, don't fret. In this tutorial
we will use the [TPM simulator]. Just download the source code and build it.

{% highlight bash %}
wget https://downloads.sourceforge.net/project/ibmswtpm2/ibmtpm1332.tar.gz
mkdir ibmtpm1332
cd ibmtpm1332
tar -xavf ../ibmtpm1332.tar.gz
cd src
make
# TODO install i.e. symlink to /usr/local/bin or change PATH
{% endhighlight %}

Now we install the other dependencies:

{% highlight bash %}
# TODO
{% endhighlight %}

## Hello World

First, let's run the TPM simulator. This will open two TCP sockets. Keep in mind
that we can use [Wireshark] to capture the traffic if needed.

 * `localhost:2321`: listening for TPM commands
 * `localhost:2322`: listening for platform commands

{% highlight bash %}
tpm_server
{% endhighlight %}

Now, let's run the resource manager which will connect to the TPM simulator.
Note that is has to be started as user `tss` for security reasons. Internally,
the library `libtss2-tcti-tabrmd.so` will be used to send commands to the
resource manager.

{% highlight bash %}
sudo -u tss tpm2-abrmd --tcti=mssim
{% endhighlight %}

Now we are ready to call our first TPM tool! After each TPM *power cycle* (e.g.
after a reboot of the PC), we need to issue the command `tpm2_startup`. If the
TPM is already ready, we will get an error 0x00000100 which we can simply
ignore.

{% highlight bash %}
tpm2_startup -c
{% endhighlight %}

The next TPM coommand is `tpm2_getrandom` which will give us a specified number
of random bytes. These bytes are [truly] random.

{% highlight bash %}
tpm2_getrandom --hex 8
{% endhighlight %}

## Creating Objects

All TPM 2.0 *entities* are referenced with a *handle* (e.g. `0x8000001`). A
subset of these entities are the so-called *objects*: keys or data. How to
create and use objects is covered in this section.

### Hierarchies

Before we create an object, it's important to understand the concept of
hierarchies. The TPM 2.0 has 4 hierarchies, they are:

 * **platform** - used by firmware and the OS.
 * **owner** - used by the owner (this is the one we will be using).
 * **endorsement** - used for privacy sensitive keys, these keys are known to
                     come from a valid TPM and platform.
 * **null** - reset on every boot, good for transient session keys.

Each hierarchy is essentially a static seed value (with the exception of the
null hierarchy, which is a new seed on every boot). This seed is used to create
one or more primary objects. Since hierarchies are also TPM entities, they can
be referenced by handle, as well.

Every hierarchy has its own **authorization value** which is basically a
password. We will dive into authorization later.

A note on terminology. The TPM supports both symmetric encryption (such as AES), and
assymetric (or public-key) encryption (like RSA). Somewhat confusingly, in both cases
the TPM refers to the key material simply as "key". Even when, as with RSA,
it would more correct to use the term key-*pair*. For example, `tpm2_createprimary`
is described as the command for creating a "Primary Key", but by default, this command
actually creates an RSA key-pair. Keep this mind as you read on.

### Primary Objects

Under a hierarchy, one can create one or more primary objects. These objects do
not persist across reboots by default. However, a primary object created with
the same inputs under a given hierarchy will produce the same exact key (with
the exception of the `null` hierarchy, the seed changes each boot). Primary
objects have special attributes and thus are not intended for general purpose
application. To add a primary object to a hierarchy requires the hierarchy
authorization value.

Usually, one of the first thing to do after the TPM startup is creating a
primary key. The following command creates an primary RSA key under the owner
hierarchy (`-C o`) and saves the key context in the file `primary.ctx`. Note
that this key has no parent and thus cannot exist outside the TPM.

{% highlight bash %}
# Create primary key
tpm2_createprimary -C o -c primary.ctx
{% endhighlight %}

You might ask yourself why we did not need to authorize to the owner hierarchy
(i.e. pass the password). By default the auth. value of the owner hierarchy is
empty for the TPM simulator. If there was a non-empty password, we would have
passed `-P <password>`.

### Creating an Object

As you might have guessed, a hierarchy is a hierarchy of objects. That means
each child key is encrypted (i.e. *wrapped*) by its parent key. Creating parent
objects requires specific *attributes* which will be covered shortly.

{% highlight bash %}
# Create parent key
tpm2_create -C primary.ctx -a 'fixedtpm|fixedparent|sensitivedataorigin|userwithauth|restricted|decrypt' -c parent.ctx

# Create child key
tpm2_create -C parent.ctx -c child.ctx
{% endhighlight %}

We can also create data objects. Again, they are encrypted (i.e. *sealed*) by
the parent object. We can specify either a file (`-i <file>`) or reading from
[stdin] (`-i-`). Note that data objects are indeed a part of the hierarchy they
were created in.

{% highlight bash %}
# Create sealed data blob
echo "my sealed data" | tpm2_create -C parent.ctx -i- -c data.ctx
{% endhighlight %}

To recover this data we use `tpm2_unseal`.

{% highlight bash %}
# Unseal data blob (i.e decrypt using parent context) data blob
tpm2_unseal -c data.ctx
{% endhighlight %}

I explained earlier that each entity inside the TPM is represented by a handle.
Well, the resource manager handles saving and loading object contexts (the data
in the `.ctx` files) for us. That means we do not care about object handles.
However, if we want, we can display information such as the internally used
handle (and e.g. the hierarchy) of an object using `tpm2_print`.

{% highlight bash %}
tpm2_print -t TPMS_CONTEXT data.ctx
{% endhighlight %}

### Object Attributes

You probably noticed that the the commands to generate objects
(`tpm2_createprimary`, `tpm2_create`) print some information about the generated
entity.

{% highlight bash %}
...
attributes:
  value: fixedtpm|fixedparent|sensitivedataorigin|userwithauth|restricted|decrypt
  raw: 0x30072
type:
  value: rsa
  raw: 0x1
exponent: 0x0
bits: 2048
...
{% endhighlight %}

In this example we generated an RSA key with a length of 2048 bits.
Additionally we see a number of attributes. The most important attributes are:

 * **fixedtpm** - key cannot be duplicated at all
 * **fixedparent** - key cannot be duplicated to a different parent
 * **sensitivedataorigin** - sensitive data (private key) is/will be generated
                             by the TPM
 * **restricted** - key can only operate on special data (needed for parent keys)
 * **decrypt** - the private key can be used to decrypt (needed for parent keys)
 * **sign** - the private key can be used to sign data
 * **noda** - multiple failed authorization attempts will not result in the
              lockout mode (see *Dictionary Attack*)

Note: if the key is *restricted*, either *decrypt* or *sign* (but not both) must
be set. 

## Importing Keys

Often, you want to import an externally generated key into the TPM. This can
be achieved by using the `tpm2_import` command.

{% highlight bash %}
# Generate key-pair externally
openssl genrsa -out key.pem 2048
openssl rsa -in key.pem -pubout -out key_pub.pem

# Create primary key
tpm2_createprimary -C o -c primary.ctx

# Import external key(-pair). Creates Primary Key encrypted public/private blobs on-disk.
tpm2_import -G rsa -i key.pem -C primary.ctx -r import_key_priv.bin -u import_key_pub.bin
# Load the new key(-pair) from the enecrypted blobs. We must pass in primary.ctx in order
# to decrypt them.
tpm2_load -C primary.ctx -r import_key_priv.bin -u import_key_pub.bin -c key.ctx
{% endhighlight %}

Alternatively, an externally generated key(-pair) can be loaded without being encrypted
(wrapped) by a parent key. Usually, this key is associated with the *null*
hierarchy (`-C n`). If there is no *sensitive portion* (e.g. only a public key),
the key can be associated with another hierarchy. However, it will never be
wrapped by a parent key.

{% highlight bash %}
# Generate key-pair externally
openssl genrsa -out key.pem 2048
openssl rsa -in key.pem -pubout -out key_pub.pem

# Load the key-pair directly, without wrapping.
tpm2_loadexternal -C n -G rsa -r key.pem -c key.ctx
{% endhighlight %}

## Signature Generation and Verification

Signing using an asymmetric key is rather easy. The *sign* attribute of the key must be set during creation.
However, *sign* is part of the default attributes for `tpm2_create`.

### TPM only

{% highlight bash %}
# Create primary key and signing key
tpm2_createprimary -C o -c primary.ctx
tpm2_create -C primary.ctx -c key.ctx -u key.pub

# Creating file with the data to be signed
echo "The data to be signed" > plain.txt

# Create signature
tpm2_sign -c key.ctx -o signature.bin plain.txt
{% endhighlight %}

Now the signature can be verified using the same key.

{% highlight bash %}
tpm2_verifysignature -c key.ctx -m plain.txt -s signature.bin
{% endhighlight %}

### Using OpenSSL

By default, the signature written to the specified file in a specific format
called `tss`. To be able to verify this signature using other means such as
[OpenSSL], we need to specify another format: `-f plain`. Additionally, to be
able to use the public key with OpenSSL, it needs to be saved in [PEM] format.

Signing with the TPM, verifying with OpenSSL:

{% highlight bash %}
# Create signature
tpm2_sign -c key.ctx -o signature.bin -f plain -g sha256 plain.txt

# Save public key in PEM format
tpm2_readpublic -c key.ctx -f pem -o key_pub.pem

# Verify signature
openssl dgst -verify key_pub.pem -sha256 -signature signature.bin plain.txt
{% endhighlight %}

Of course we can also sign with OpenSSL and verify using the TPM. Note that in
this case loading the public portion of the key into the TPM is enough for the
signature verification. To spice things up a little bit, we use Elliptic Curve
Cryptography (ECC) keys.

Signing with OpenSSL, verifying with the TPM:

{% highlight bash %}
# Generate key-pair externally
openssl ecparam -name prime256v1 -genkey -out key.pem
openssl ec -in key.pem -pubout -out key_pub.pem

# Create signature
openssl dgst -sign key.pem -sha256 -out signature.bin plain.txt

# Load public key
tpm2_loadexternal -G ecc -u key_pub.pem -c key.ctx

# Verify signature
tpm2_verifysignature -c key.ctx -m plain.txt -g sha256 -f ecdsa -s signature.bin
{% endhighlight %}

## Persistance

By default, every key we created (or imported) is a *transient* object. This
means it's in the TPM's volatile memory and will be lost after a reboot (or a
TPM reset). To take advantage of the TPM's non-volatile memory (NV), we have to
instruct the TPM to store it persistently. This enables us to use keys without
having them stored on the computer's file system (because the file system is not
available in early stages of boot or because the platform is a microcontroller
without any file system at all).

{% highlight bash %}
# Create primary key and child key
tpm2_createprimary -C o -c primary.ctx
tpm2_create -C primary.ctx -c key.ctx

# Persist the transient key
tpm2_evictcontrol -c key.ctx 0x81000000

# List the handles of all persistent objects
tpm2_getcap handles-persistent

# Evict persistent key
tpm2_evictcontrol -c 0x81000000
{% endhighlight %}

In this example we already chose a persistent handle: `0x81000000`. This handle
is used to reference the persistent onject. If we did not specify a handle, the
TPM would have assigned a handle.

We also learned that there is a way of listing the handles of all persistent
objects using the `tpm2_getcap` (get capability) tool.

## NV Space

{% highlight bash %}
# Create primary key and child key
tpm2_nvdefine -Q  1 -C o -s 32 -a "ownerread|policywrite|ownerwrite"

echo "please123abc" > nv.test_w

tpm2_nvwrite -Q   $nv_test_index -C o -i nv.test_w

tpm2_nvread -Q  1 -C o -s 32 -o 0
{% endhighlight %}

## Authorization

Until now, we did not worry about authorization. Obviously we do not want our
keys to be accessible by anyone. To protect our secrets, the TPM offers three
kind of authorization "sessions":

  * password session
  * HMAC session
  * policy session

### Password Session

Password authorizations are the simplest authorizations. Every object has an
auth. value (i.e. password) which is by default empty. If the auth. value is specified during
object creation or changed afterwards, the entity can only be used with
authorization.

The TPM provides a special password session which does not need to be started
and does not maintain any state. It is used to send a password in plaintext to
the TPM alongside a command. For the tpm2-tools, to authorize an entity you
usually pass `-p <password>`. If an object's parent is to be authorized, an
uppercase `-P <password>` is passed.

The auth. value can be set during creation of a TPM entity or later on.

Commands to set the authorization value during creation:

 * `tpm2_createprimary` for primary objects
 * `tpm2_create` for any other object
 * `tpm2_nvdefine` for NV indices

command to set the authorization value later:

 * `tpm2_changeauth` for transient and persistent objects, hierarchies and NV
                     indices

In the following example, the authorization of a key object is demonstrated.
TPM objects need to be loaded after a call of `tpm2_changeauth`.

{% highlight bash %}
# Create primary key
tpm2_createprimary -c primary.ctx

# Create child key with password 123456
tpm2_create -C primary.ctx -p 123456 -c key.ctx -u key_pub.bin -p 123456

# Use the key to sign some data
echo "The data to be signed" > plain.txt
tpm2_sign -c key.ctx -o signature.bin -p 123456 plain.txt

# Change password to 000000 and load
tpm2_changeauth -c key.ctx -C primary.ctx -r key_priv.bin -p 123456 000000
tpm2_load -C primary.ctx -r key_priv.bin -u key_pub.bin -c key.ctx

# Use the key again
tpm2_sign -c key.ctx -o signature.bin -p 000000 plain.txt
{% endhighlight %}

### HMAC Sessions



## Behind the scenes

// TODO

The TPM's memory is quite constraint. To work around this, in most cases the TPM
will not store the keys in its own memory, but instead return encrypted blobs
of the keys when they are created, these blobs are called *key context*. Whenever
we need to use such a key, we can send this *key context* blob the TPM,
specifying the Parent Key which was used, and have the TPM internally decrypt it
and load into memory for transient use. We can of course also request for certain keys
to be stored in nvram, as shown above.

To eliminate the need to manually load these context objects every time,
we can use a  "Broker and Resource Manager" (ABRM). They are in charge
of loading these contexts on demand, transperantly, so from the application's
point-of-view, all the keys are always available to the TPM.
There are two notable ABRM implementations:

 * The in-linux-kernel resource manager which is part of the [TPM device
   driver]. Typically, the driver provides a character device `/dev/tpmrm0`.
 * The user space [tpm2-abrmd] which can be used with the TPM simulator and
   offers some features such as easy logging etc.


{% highlight bash %}
# TODO
{% endhighlight %}


[tpm2-tools]: https://github.com/tpm2-software/tpm2-tools
[certainly contains a TPM 2.0]: https://docs.microsoft.com/en-us/windows-hardware/design/minimum/minimum-hardware-requirements-overview#37-trusted-platform-module-tpm
[TPM simulator]: https://sourceforge.net/projects/ibmswtpm2/
[TPM device driver]: http://git.infradead.org/users/jjs/linux-tpmdd.git
[tpm2-abrmd]: https://github.com/tpm2-software/tpm2-abrmd
[Wireshark]: https://www.wireshark.org/
[truly]: https://en.wikipedia.org/wiki/Hardware_random_number_generator
[stdin]: https://en.wikipedia.org/wiki/Standard_streams#Standard_input_(stdin)
[OpenSSL]: https://www.openssl.org/
[PEM]: https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail

