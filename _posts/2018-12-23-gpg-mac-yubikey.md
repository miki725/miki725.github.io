---
layout: post
title:  "GPG on a Mac with a Yubikey"
subtitle: ""
date: 2018-12-23
categories: [crypto]
---

* TOC
{:toc}

This tutorial is a step-by-step guide on how to setup your GPG key
on a Mac, transfer it to a [Yubikey] for security and later use it for [Git Signing] as well as SSH authentication.

## Disclaimer

I am pretty new to GPG management myself so the steps in this post
might not be the most efficient/effective but they did work for me.
Just trying to share it with others as I had to combine multiple sources
to make everything work. Also documenting all the steps
for myself for future use since Im sure I will forget everything
in about a month.

## Prerequisites

You can use [Homebrew] to install all necessary packages.

{% highlight bash %}
❯❯❯ brew install \
    gpg \
    paperkey \
    qrencode \
    zbar \
    pinentry-mac \
    homebrew/cask-drivers/yubico-yubikey-manager
{% endhighlight %}

Note that `gpg2` is required however `brew`'s recipy for `gpg` installs `gpg2` by default.
You can check that by looking up recipy information:

{% highlight bash %}
❯❯❯ brew info gpg | head -n1
gnupg: stable 2.2.12 (bottled)
{% endhighlight %}

GPG will not allow to enter passwords unless TTY is configured:

{% highlight bash %}
❯❯❯ export GPG_TTY=$(tty)
{% endhighlight %}

## Generate keys

GPG has a notion of a keyring.
Keyring as the name implies is very similar to a physical keyring
which holds multiple of your keys where each key is used for a different purpose - encryption, signing or authenticating your identity.
In addition keys can be derived from a master key.
This property is useful for security reasons as it allows to revoke 
individual keys without needing to revoke the complete master key.
Speaking of security, its also a good practice to expire the keys in few
years just in case any issues are found with RSA or the key gets compromised.

### Master key

The following will generate the master key.
Note that even though EC crypto is pretty awesome, it is currently 
unsupported by any of the Yubikeys hence RSA keys need to be used.
In this example we will create 2048 byte RSA keys expiring in 5 years.
Newest Yubikeys (4 and above) support 4096 byte keys.

{% highlight bash %}
❯❯❯ gpg --expert --full-generate-key
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
   (9) ECC and ECC
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (13) Existing key
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 2048
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 5y
Key expires at Sat Dec 23 01:09:57 2023 EST
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: GPG Test
Email address: test@example.com
Comment:
You selected this USER-ID:
    "GPG Test <test@example.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: key 48621A68A9AD7551 marked as ultimately trusted
gpg: directory '/Users/miki725/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/Users/miki725/.gnupg/openpgp-revocs.d/ED5CA76BDA3809828BB5646A48621A68A9AD7551.rev'
public and secret key created and signed.

Note that this key cannot be used for encryption.  You may want to use
the command "--edit-key" to generate a subkey for this purpose.
pub   rsa2048 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
uid                      GPG Test <test@example.com>
{% endhighlight %}

You can verify master key was successfully created:

{% highlight bash %}
❯❯❯ gpg --list-keys --keyid-format LONG
/Users/miki725/.gnupg/pubring.kbx
---------------------------------
pub   rsa2048/48621A68A9AD7551 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
uid                 [ultimate] GPG Test <test@example.com>
{% endhighlight %}

The created master key can either be referred by its id `48621A68A9AD7551`
or by the email address `test@example.com`.

### Subkeys

Now subkeys can be added for encryption, signing and authenticating.
Encryption and signing keys are pretty straight forward.
Authentication key requires to customize key's capabilities (see below).

{% highlight bash %}
❯❯❯ gpg --expert --edit-key test@example.com
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
[ultimate] (1). GPG Test <test@example.com>

gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 6
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 2048
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 5y
Key expires at Sat Dec 23 01:20:19 2023 EST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
[ultimate] (1). GPG Test <test@example.com>

gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 2048
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 5y
Key expires at Sat Dec 23 01:20:47 2023 EST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
[ultimate] (1). GPG Test <test@example.com>

gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 8

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? s

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? e

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions:

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? a

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Authenticate

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 2048
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 5y
Key expires at Sat Dec 23 01:21:51 2023 EST
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> save
{% endhighlight %}

You can verify subkeys were created by listing all keys:

{% highlight bash %}
❯❯❯ gpg --list-keys --keyid-format LONG
/Users/miki725/.gnupg/pubring.kbx
---------------------------------
pub   rsa2048/48621A68A9AD7551 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
uid                 [ultimate] GPG Test <test@example.com>
sub   rsa2048/7934F75F8D18C1DD 2018-12-24 [E] [expires: 2023-12-23]
sub   rsa2048/FF46BE567B8D4C88 2018-12-24 [S] [expires: 2023-12-23]
sub   rsa2048/2A136E849931EB24 2018-12-24 [A] [expires: 2023-12-23]
{% endhighlight %}

## Backup Master Key

### Encrypted Archive

Since subkeys provide all the necessary functionality, 
master key is no longer necessary.
Therefore master key can be backed up and removed from the keyring
for security reasons.
The backup should be kept in a secure location
like on an encrypted disk or paper in a safe.

Since we will be exporting sensitive information it's not a good
idea to save any sensitive files on disk. Ramdisk to the rescue!

{% highlight bash %}
>>> diskutil erasevolume HFS+ 'ramdisk' $(hdiutil attach -nomount ram://262144)
Started erase on disk2
Unmounting disk
Erasing
Initialized /dev/rdisk2 as a 128 MB case-insensitive HFS Plus volume
Mounting disk
Finished erase on disk2 ramdisk
{% endhighlight %}

Now both private and public keys can be written to ramdisk.
In addition its a good idea to generate revocation certificate
in case master key will need to be revoked without having access
to master key.

{% highlight bash %}
❯❯❯ gpg --export test@example.com > \
        /Volumes/ramdisk/public.key
❯❯❯ gpg --export-secret-key test@example.com > \
        /Volumes/ramdisk/private.key
❯❯❯ gpg --gen-revoke test@example.com > \
        /Volumes/ramdisk/revoke.cert

sec  rsa2048/48621A68A9AD7551 2018-12-24 GPG Test <test@example.com>

Create a revocation certificate for this key? (y/N) y
Please select the reason for the revocation:
  0 = No reason specified
  1 = Key has been compromised
  2 = Key is superseded
  3 = Key is no longer used
  Q = Cancel
(Probably you want to select 1 here)
Your decision? 1
Enter an optional description; end it with an empty line:
>
Reason for revocation: Key has been compromised
(No description given)
Is this okay? (y/N) y
ASCII armored output forced.
Revocation certificate created.

Please move it to a medium which you can hide away; if Mallory gets
access to this certificate he can use it to make your key unusable.
It is smart to print this certificate and store it away, just in case
your media become unreadable.  But have some caution:  The print system of
your machine might store the data and make it available to others!
{% endhighlight %}

For convenience and security all files can be combined into an encrypted archive.

{% highlight bash %}
❯❯❯ tar cz -C /Volumes/ramdisk/ \
        private.key \
        public.key \
        revoke.cert | \
    gpg --cipher-algo AES256 \
        -z 0 \
        --symmetric \
        --armor > \
    /Volumes/ramdisk/master.tar.gz.gpg
{% endhighlight %}

The archive can be safely backed up, hopefully to an encrypted storage
and stored offline.

### Paper

Alternatively there are couple of methods of printing the key to paper.
[Paperkey] seems to be the most mature solution.
It only extracts the secret bits out of the key hence no need
to print the complete private key as the secret keys can be combined
with public key to restore private key.
Still, secret bits are larger than QR code can accommodate
hence they need to be split to multiple files.

Below will:

1. extracts secret bits out of the private key
2. combine secret bits with revocation certificate to an archive
3. encrypt archive for safety
4. splits archive into 2.5K chunks to accommodate QR code max limit
5. QR encodes each chunk which generates PNG files

{% highlight bash %}
❯❯❯ cat /Volumes/ramdisk/private.key | \
    paperkey --output-type=raw > \
    /Volumes/ramdisk/secret.gpg
❯❯❯ tar cz -C /Volumes/ramdisk/ \
        secret.gpg \
        revoke.cert | \
    gpg --cipher-algo AES256 \
        -z 0 \
        --symmetric \
        --armor > \
    /Volumes/ramdisk/paper.tar.gz.gpg
❯❯❯ split -C 2500 \
        /Volumes/ramdisk/paper.tar.gz.gpg \
        /Volumes/ramdisk/paper.tar.gz.gpg-
❯❯❯ for file in /Volumes/ramdisk/paper.tar.gz.gpg-??;
    do qrencode -d 600 -r $file -o $file.png;
    done
❯❯❯ ls /Volumes/ramdisk/paper.tar.gz.gpg-??.png
/Volumes/ramdisk/paper.tar.gz.gpg-aa.png
/Volumes/ramdisk/paper.tar.gz.gpg-ab.png
/Volumes/ramdisk/paper.tar.gz.gpg-ac.png
{% endhighlight %}

PNGs can be safely printed and stored in a safe place.

## Restore Master Key from Paper

Master key can be fully restored either from master archive
or from printed QR codes.
This will demonstrate how to restore from QR codes.

{% highlight bash %}
❯❯❯ for file in /Volumes/ramdisk/paper.tar.gz.gpg-??.png;
    do zbarimg -q --raw $file | \
        head -n -1 > \
        "$file.decoded";
    done
❯❯❯ cat /Volumes/ramdisk/paper.tar.gz.gpg-??.png.decoded > \
    /Volumes/ramdisk/paper.tar.gz.gpg.restored
❯❯❯ gpg \
        --cipher-algo AES256 \
        -z 0 \
        --decrypt /Volumes/ramdisk/paper.tar.gz.gpg.restored | \
    tar zxv
gpg: AES256 encrypted data
gpg: encrypted with 1 passphrase
secret.gpg
revoke.cert
❯❯❯ paperkey \
        --pubring /Volumes/ramdisk/public.key \
        --secrets /Volumes/ramdisk/secret.gpg > \
    /Volumes/ramdisk/private.key.restored
{% endhighlight %}

To simulate complete loss of GPG keyring, GPG home directory can be deleted.
Then we can restore the master key.

{% highlight bash %}
❯❯❯ rm -rf ~/.gnupg
❯❯❯ gpg --import /Volumes/ramdisk/private.key.restored
❯❯❯ gpg --list-secret-keys --keyid-format LONG
/Users/miki725/.gnupg/pubring.kbx
---------------------------------
sec   rsa2048/48621A68A9AD7551 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
uid                 [ unknown] GPG Test <test@example.com>
ssb   rsa2048/7934F75F8D18C1DD 2018-12-24 [E] [expires: 2023-12-23]
ssb   rsa2048/FF46BE567B8D4C88 2018-12-24 [S] [expires: 2023-12-23]
ssb   rsa2048/2A136E849931EB24 2018-12-24 [A] [expires: 2023-12-23]
{% endhighlight %}

Only issue is that the key is not trusted so it needs to be edited:

{% highlight bash %}
❯❯❯ gpg --edit-key test@example.com
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

sec  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: unknown       validity: unknown
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ unknown] (1). GPG Test <test@example.com>

gpg> trust
sec  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: unknown       validity: unknown
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ unknown] (1). GPG Test <test@example.com>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

sec  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: unknown
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ unknown] (1). GPG Test <test@example.com>
Please note that the shown key validity is not necessarily correct
unless you restart the program.

gpg> save
{% endhighlight %}

Master key is restored now  which can be verified by listing all secret keys.

{% highlight bash %}
❯❯❯ gpg --list-secret-keys --keyid-format LONG
/Users/miki725/.gnupg/pubring.kbx
---------------------------------
sec   rsa2048/48621A68A9AD7551 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
uid                 [ultimate] GPG Test <test@example.com>
ssb   rsa2048/7934F75F8D18C1DD 2018-12-24 [E] [expires: 2023-12-23]
ssb   rsa2048/FF46BE567B8D4C88 2018-12-24 [S] [expires: 2023-12-23]
ssb   rsa2048/2A136E849931EB24 2018-12-24 [A] [expires: 2023-12-23]
{% endhighlight %}

## Remove Master Key

Like mentioned previously, master key is very sensitive
hence it should be deleted locally after secure backup is made.
With newest versions of GPG that is relatively easy
as private keys are stored in GPG home directory.

{% highlight bash %}
❯❯❯ ls ~/.gnupg/private-keys-v1.d/
16567857EE938FED11A47969F40451BF014993F4.key
8254BE2A4A37393C9E20CD6F7E42E4F37909D09E.key
781682C203E3D31D9FFF964FFD0A398689B4CBBF.key
A72DAE0F8F392E69F5EE4ED60361F490EBAB31D1.key
{% endhighlight %}

The key names are keygrip ids.
To correlate the ids you can list all keys with their keygrip ids.

{% highlight bash %}
❯❯❯ gpg --list-keys --keyid-format LONG --with-keygrip
/Users/miki725/.gnupg/pubring.kbx
---------------------------------
pub   rsa2048/48621A68A9AD7551 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
      Keygrip = 16567857EE938FED11A47969F40451BF014993F4
uid                 [ultimate] GPG Test <test@example.com>
sub   rsa2048/7934F75F8D18C1DD 2018-12-24 [E] [expires: 2023-12-23]
      Keygrip = A72DAE0F8F392E69F5EE4ED60361F490EBAB31D1
sub   rsa2048/FF46BE567B8D4C88 2018-12-24 [S] [expires: 2023-12-23]
      Keygrip = 781682C203E3D31D9FFF964FFD0A398689B4CBBF
sub   rsa2048/2A136E849931EB24 2018-12-24 [A] [expires: 2023-12-23]
      Keygrip = 8254BE2A4A37393C9E20CD6F7E42E4F37909D09E
{% endhighlight %}

`16567857EE938FED11A47969F40451BF014993F4` is the master key
so that key can be removed.

{% highlight bash %}
❯❯❯ rm ~/.gnupg/private-keys-v1.d/16567857EE938FED11A47969F40451BF014993F4.key
{% endhighlight %}

You can verify master key is removed by listing all keys.

{% highlight bash %}
❯❯❯ gpg --list-secret-keys --keyid-format LONG
/Users/miki725/.gnupg/pubring.kbx
---------------------------------
sec#  rsa2048/48621A68A9AD7551 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
uid                 [ultimate] GPG Test <test@example.com>
ssb   rsa2048/7934F75F8D18C1DD 2018-12-24 [E] [expires: 2023-12-23]
ssb   rsa2048/FF46BE567B8D4C88 2018-12-24 [S] [expires: 2023-12-23]
ssb   rsa2048/2A136E849931EB24 2018-12-24 [A] [expires: 2023-12-23]
{% endhighlight %}

Note `sec` is displayed as `sec#` now. The `#` signifies private key is not available.

## Key Server

Public key can be submitted to a keyserver so that other people can find your
public key to be able to communicate with you.
I like [keybase.io].
After uploading the public key to a keyserver, you should be able
to get a URL of where your public key can be downloaded.
Mine is for example:

[https://keybase.io/miki725/pgp_keys.asc](https://keybase.io/miki725/pgp_keys.asc)

This URL will be used later.

## Yubikey

### Better Defaults

Before subkeys can be moved to a Yubikey, Yubikey itself needs to be configured
to support smart card mode.
It can be done with CLI however I prefer the official
Yubico "Yubikey Manager" as it will forbid me from messing anything up.
Yubikey Manager was installed with `brew` above
so simply open the application.

![Yubikey Manager enabling CCID mode](/assets/images/2018-12-23-gpg-mac-yubikey/yubikey-manager.png)

After enabling CCID mode, you can verify its working
with GPG:

{% highlight bash %}
❯❯❯ gpg --card-status
Reader ...........: Yubico Yubikey 4 OTP U2F CCID
Application ID ...: XXXXX
Version ..........: 2.1
Manufacturer .....: Yubico
Serial number ....: XXXXX
Name of cardholder: [not set]
Language prefs ...: [not set]
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
{% endhighlight %}

Before transferring keys, it's a good idea to change
a few card configurations:

* change default pins from their default
* set reset code
* force card to prompt for pin when signing
* name of cardholder
* language preferences
* url of public key
* your login username

Default pin is `123456` and admin pin is `12345678`.

{% highlight bash %}
❯❯❯ gpg --edit-card

Reader ...........: Yubico Yubikey 4 OTP U2F CCID
Application ID ...: XXXXX
Version ..........: 2.1
Manufacturer .....: Yubico
Serial number ....: XXXXX
Name of cardholder: [not set]
Language prefs ...: [not set]
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]

gpg/card> admin
Admin commands are allowed

gpg/card> passwd
gpg: OpenPGP card no. XXXXX detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 4
Reset Code set.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? q

gpg/card> forcesig

gpg/card> name
Cardholder's surname: Shubernetskiy
Cardholder's given name: Miroslav

gpg/card> lang
Language preferences: en

gpg/card> url
URL to retrieve public key: https://keybase.io/miki725/pgp_keys.asc

gpg/card> login
Login data (account name): miki725

gpg/card> quit
{% endhighlight %}

You can verify all settings were applied by checking card status again.

{% highlight bash %}
❯❯❯ gpg --card-status
Reader ...........: Yubico Yubikey 4 OTP U2F CCID
Application ID ...: XXXXX
Version ..........: 2.1
Manufacturer .....: Yubico
Serial number ....: XXXXX
Name of cardholder: Miroslav Shubernetskiy
Language prefs ...: en
Sex ..............: unspecified
URL of public key : https://keybase.io/miki725/pgp_keys.asc
Login data .......: miki725
Signature PIN ....: forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
{% endhighlight %}

### Touch Required

In addition Yubikey supports a feature which forces you to touch Yubikey
anytime it tries to interact with private keys stored on the card.
This protects you against malicious scripts attempting to use
GPG without your consent.

Full docs for that are located on [Yubico Edit Card] page.
[yubitouch] repo contains the script which allows to change touch mode of a Yubikey.
By default the script does not work on a Mac since it requires a different
`pinentry` program unless you change your default GPG config.
I opted to change the script instead to use `pinentry-mac`
previously installed with `brew`.

{% highlight bash %}
❯❯❯ curl -s https://raw.githubusercontent.com/a-dma/yubitouch/master/yubitouch.sh | \
    sed 's/pinentry/pinentry-mac/' | \
    bash -s sig on
All done!
{% endhighlight %}

### Transfer keys

Now individual keys can be transferred to the Yubikey.
Note that each subkey needs to be moved independently.

{% highlight bash %}
❯❯❯ gpg --edit-key test@example.com
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret subkeys are available.

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> key 1

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb* rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb* rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> key 1

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> key 2

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb* rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb* rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> key 2

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb  rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> key 3

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb* rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3

pub  rsa2048/48621A68A9AD7551
     created: 2018-12-24  expires: 2023-12-23  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa2048/7934F75F8D18C1DD
     created: 2018-12-24  expires: 2023-12-23  usage: E
ssb  rsa2048/FF46BE567B8D4C88
     created: 2018-12-24  expires: 2023-12-23  usage: S
ssb* rsa2048/2A136E849931EB24
     created: 2018-12-24  expires: 2023-12-23  usage: A
[ultimate] (1). GPG Test <test@example.com>

gpg> save
{% endhighlight %}

You can verify all keys were moved by checking card status again.

{% highlight bash %}
❯❯❯ gpg --card-status
Reader ...........: Yubico Yubikey 4 OTP U2F CCID
Application ID ...: XXXXX
Version ..........: 2.1
Manufacturer .....: Yubico
Serial number ....: XXXXX
Name of cardholder: Miroslav Shubernetskiy
Language prefs ...: en
Sex ..............: unspecified
URL of public key : https://keybase.io/miki725/pgp_keys.asc
Login data .......: miki725
Signature PIN ....: forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: 658F 79F5 F344 AF38 A1BB  E941 FF46 BE56 7B8D 4C88
      created ....: 2018-12-24 06:20:30
Encryption key....: 607C BF0E 6082 25D0 E91E  923D 7934 F75F 8D18 C1DD
      created ....: 2018-12-24 06:20:01
Authentication key: 2957 0C23 4DFA EDD1 A23D  1183 2A13 6E84 9931 EB24
      created ....: 2018-12-24 06:20:53
General key info..: sub  rsa2048/FF46BE567B8D4C88 2018-12-24 GPG Test <test@example.com>
sec#  rsa2048/48621A68A9AD7551  created: 2018-12-24  expires: 2023-12-23
ssb>  rsa2048/7934F75F8D18C1DD  created: 2018-12-24  expires: 2023-12-23
                                card-no: 0006 08753000
ssb>  rsa2048/FF46BE567B8D4C88  created: 2018-12-24  expires: 2023-12-23
                                card-no: 0006 08753000
ssb>  rsa2048/2A136E849931EB24  created: 2018-12-24  expires: 2023-12-23
                                card-no: 0006 08753000
{% endhighlight %}

You can attempt to sign something which will require you to enter your PIN
as well as touch the Yubikey.

{% highlight bash %}
❯❯❯ echo hello | gpg --armor --sign
-----BEGIN PGP MESSAGE-----

owGbwMvMwMH4321fWHWvTwfjaZ4khhhFz3kZqTk5+VydjMYsDIwcDLJiiiyp/ZVf
P7ust1i4+6UjTDkrE0gtAxenAEzkyDkOhhN2C7xXn1L996n0lucnwYwPh6WEd72y
U3Bq5nne2/rSac7HLhmHC9+fSE6dJ8gXUBip9ydM2e6Aesdbs1PeZYL3p65c8UA3
ffeUwKWrZnSVlE6+v/C9ZGaqhv0snbPfzv+IsPH5JO+QpDRlSub9pWvu2T0MX66u
rj7159XXdv7L259xrNpSYHnm37brKSmuFb7a0wXlTFYIr+E9Fbl9seKFnxHLu8yn
1WzoL1Kz1W+ynvn4tJ7m3X3cWUs9kz//nLlZX/l4w8WqO/IHbRt72Oo5D+dICH5z
tIycttz500fh/Eq3B8zqtdyCS3jcjlptfLLbIPTR3NTQIx43Mlk+cL2YJKfAb+Jq
aiPHtHxyJAA=
=GTbX
-----END PGP MESSAGE-----
{% endhighlight %}

If you remove Yubikey you should not be able to sign anything anymore.

{% highlight bash %}
❯❯❯ echo hello | gpg --armor --sign
┌─────────────────────────────────────────────┐
│ Please insert the card with serial number:  │
│                                             │
│ XXXX XXXXXXXX                               │
│                                             │
│      <OK>                       <Cancel>    │
└─────────────────────────────────────────────┘
{% endhighlight %}

## Git Signing

### Config

Before we continue you need to find the signing key id.

{% highlight bash %}
❯❯❯ gpg --list-keys --keyid-format LONG
/Users/miki725/.gnupg/pubring.kbx
---------------------------------
pub   rsa2048/48621A68A9AD7551 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
uid                 [ultimate] GPG Test <test@example.com>
sub   rsa2048/7934F75F8D18C1DD 2018-12-24 [E] [expires: 2023-12-23]
sub   rsa2048/FF46BE567B8D4C88 2018-12-24 [S] [expires: 2023-12-23]
sub   rsa2048/2A136E849931EB24 2018-12-24 [A] [expires: 2023-12-23]
{% endhighlight %}

In this case the key id is `FF46BE567B8D4C88`.

Now you can configure a couple of git parameters to make it all work.

{% highlight bash %}
❯❯❯ git config --global commit.gpgsign true
❯❯❯ git config --global gpg.program /usr/local/bin/gpg
❯❯❯ git config --global user.signingkey FF46BE567B8D4C88
{% endhighlight %}

### Demo

Create repo and sign first commit.

{% highlight bash %}
❯❯❯ mkdir /tmp/test
❯❯❯ cd /tmp/test/
❯❯❯ git init
Initialized empty Git repository in /private/tmp/test/.git/
❯❯❯ touch hello
❯❯❯ git add hello
❯❯❯ git commit -m 'test' -S
[master (root-commit) 149e91d] test
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 hello
{% endhighlight %}

You can verify commit was signing by checking `git log`.

{% highlight bash %}
❯❯❯ git log --show-signature
commit 149e91df86cfe412bfc54a447f4dc7945f63855e (HEAD -> master)
gpg: Signature made Mon Dec 24 16:14:40 2018 EST
gpg:                using RSA key 658F79F5F344AF38A1BBE941FF46BE567B8D4C88
gpg: Good signature from "GPG Test <test@example.com>" [ultimate]
Author: Miroslav Shubernetskiy <miroslav@miki725.com>
Date:   Mon Dec 24 16:14:40 2018 -0500

    test
{% endhighlight %}

## SSH

### GPG Agent

By default GPG agent does not support SSH so that needs to be changed.
Later it should be restarted to take effect.

{% highlight bash %}
❯❯❯ echo "enable-ssh-support" >> \
    ~/.gnupg/gpg-agent.conf
❯❯❯ gpg-agent $(gpg-connect-agent reloadagent /bye)
gpg-agent[33544]: gpg-agent running and available
{% endhighlight %}

For some reason though SSH was not honored for me.
Not sure if its a GPG bug but for now I need to explicitly update agent config.
See [mailing list](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=835394)
for more information.

{% highlight bash %}
❯❯❯ gpg-connect-agent updatestartuptty /bye > /dev/null
{% endhighlight %}

Also GPG agent needs to be told which key to use for SSH auth.
It needs keygrip id of the auth key.

{% highlight bash %}
❯❯❯ gpg --list-keys --keyid-format LONG --with-keygrip
/Users/miki725/.gnupg/pubring.kbx
---------------------------------
pub   rsa2048/48621A68A9AD7551 2018-12-24 [SC] [expires: 2023-12-23]
      ED5CA76BDA3809828BB5646A48621A68A9AD7551
      Keygrip = 16567857EE938FED11A47969F40451BF014993F4
uid                 [ultimate] GPG Test <test@example.com>
sub   rsa2048/7934F75F8D18C1DD 2018-12-24 [E] [expires: 2023-12-23]
      Keygrip = A72DAE0F8F392E69F5EE4ED60361F490EBAB31D1
sub   rsa2048/FF46BE567B8D4C88 2018-12-24 [S] [expires: 2023-12-23]
      Keygrip = 781682C203E3D31D9FFF964FFD0A398689B4CBBF
sub   rsa2048/2A136E849931EB24 2018-12-24 [A] [expires: 2023-12-23]
      Keygrip = 8254BE2A4A37393C9E20CD6F7E42E4F37909D09E
{% endhighlight %}

In this case that's `8254BE2A4A37393C9E20CD6F7E42E4F37909D09E`.

{% highlight bash %}
❯❯❯ echo 8254BE2A4A37393C9E20CD6F7E42E4F37909D09E >> \
    ~/.gnupg/sshcontrol
{% endhighlight %}

### SSH Config

Few configurations are required for SSH to work.
Main change is that ssh agent needs to be switched to use gpg agent.

{% highlight bash %}
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
export SSH_AGENT_PID=""
{% endhighlight %}

If all worked well you should see your key in SSH:

{% highlight bash %}
❯❯❯ ssh-add -l
2048 SHA256:OlsOOlrx46NgGm2l/Rmz/cCOEdJppNWD7B8fOKDfE4k cardno:XXXXX (RSA)
2048 SHA256:OlsOOlrx46NgGm2l/Rmz/cCOEdJppNWD7B8fOKDfE4k (none) (RSA)
{% endhighlight %}

The double entry is normal and is due to the Yubikey storing the same key.

To see the SSH public key you can either explicitly export it by using GPG key id.
In this case its `2A136E849931EB24`.
Alternatively you can view it in `ssh-add`.

{% highlight bash %}
❯❯❯ gpg --export-ssh-key 2A136E849931EB24
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCqzT5cnRJ2a8bxZi0CZqIkP5EYxAaRJnBwX9ZwiVaCDTYc8DupCVrrsohLhNyzIHxZMGOS7IfN5mcHGpJt8L2ZT8WqhHvJlH/46T0UyYbdiQWeA7zdEJeoQg4T1qe701K5j88cWPszzpskKT3hQbSJ28KeqjTqK8vr/DftIM+t9RqEbmiRlg78/6bqsG/KaT+ALMu7BakeDVpwzuWTcKOJFAnigxk/3Fqgjc3f2depNeOZkY0nGTHJb02bkN9zdRaeF8kTt6M3PJ+MhfZrnZoohWsr3mih538uWdgQrx4mYobRvS7XbEywDkigfKUB/t98Xr1he1QWxoSMwj4qs+xJ openpgp:0x9931EB24

❯❯❯ ssh-add -L | tail -n1
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCqzT5cnRJ2a8bxZi0CZqIkP5EYxAaRJnBwX9ZwiVaCDTYc8DupCVrrsohLhNyzIHxZMGOS7IfN5mcHGpJt8L2ZT8WqhHvJlH/46T0UyYbdiQWeA7zdEJeoQg4T1qe701K5j88cWPszzpskKT3hQbSJ28KeqjTqK8vr/DftIM+t9RqEbmiRlg78/6bqsG/KaT+ALMu7BakeDVpwzuWTcKOJFAnigxk/3Fqgjc3f2depNeOZkY0nGTHJb02bkN9zdRaeF8kTt6M3PJ+MhfZrnZoohWsr3mih538uWdgQrx4mYobRvS7XbEywDkigfKUB/t98Xr1he1QWxoSMwj4qs+xJ (none)
{% endhighlight %}

## Conclusion

Congratulations. You are done.

As a recap you did the following:

* Created GPG master key
* Made master key backup (and stored in secure location)
* Removed master key locally for security leaving only subkeys locally
* Configured Yubikey to be more secure
* Moved subkeys to Yubikey
* Configured git to use GPG signing
* Configured SSH to use GPG key

[Yubikey]: https://www.yubico.com/products/yubikey-hardware/
[Git Signing]: https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work
[Homebrew]: https://brew.sh/
[Paperkey]: https://www.jabberwocky.com/software/paperkey/
[keybase.io]: https://keybase.io
[Yubico Edit Card]: https://developers.yubico.com/PGP/Card_edit.html
[yubitouch]: https://github.com/a-dma/yubitouch