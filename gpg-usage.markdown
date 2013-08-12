# Introduction

This guide is intended to provide an experienced technical user with the information needed to use GPG as safely and effectively as possible.

The version of GPG used in this guide is `1.4.12-7` from Debian's wheezy/main repository.

## Caveats

As with many things, this guide has limitations.

The author is a Debian user, so all the specific instructions are based on Debian 7.x. If you use a different version of Debian or Debian-like OS such as Ubuntu or Mint, things might just work for you unchanged. If you’re an OSX user you might need to change something here or there. Unfortunately, the author is not able to provide advice for Windows.

This guide is targeted at a technically experienced user. It focuses on use of the terminal, not on graphical applications. Sadly, this makes it inaccessible to a lot of people. At the time of writing, GPG is not a very user-friendly piece of software, and there aren't many graphical frontends which both work well and support the advanced usage scenario described in this guide. If you find an application which works and is compatible with this guide, please get in touch: the author would love to hear from you!

If you try this guide and it doesn’t work for you, please email the author and tell them what broke! If you use OSX or Windows, you are invited to port this guide for your OS, and to let the author know what the right instructions are for you.


## License

The license for this work is [Creative Commons Attributuion-ShareAlike-3.0 USA](
https://creativecommons.org/licenses/by-sa/3.0/us). You are wecome to re-use it under that licence, but the author requests that you send changes upstream by email or pull request rather than forking outright.


## Target Setup

If you follow this guide through, you should end up with:

* a decent understanding of what your GPG client is doing some of the time,
* one or more encrypted backups of your GPG master key on removable storage,
* a smartcard (or two) with your everday keys ready to go,
* your main computer set up and ready to use GPG, but without any private keys,
* a signed certification policy and misc crypto `id.txt` document,
* plans to deal with computer compromise, loss, or other minor catastropies.


# Securely Generating Keys

## Before you Start

### Materials

You may want to get some things ready before following this guide. To follow all the steps, you'll need:

1. Your main computer: the one that you use everyday and want to use GPG on.
2. A "secure" computer, or a live disk to use with your everyday computer. See [Securing a Computer](#securing-a-computer).
3. One or more smartcard(s) and readers for them. See [Smartcard Selection](#smartcard-selection).
4. Some storage for your master key and backups: flash drive(s), or perhaps a printer.
5. A hardware random number generator, if you have one. See [What's Random?](#whats-random).


### Environment/Preparation

Cryptography is only a secure as the computer that's running it. If your private keys are stored on a computer which is compromised, the attacker can decrypt your messages, make valid signatures, or impersonate you. One of the key features of this guide is that you never end up showing your private keys to your everyday computer. Even if your everyday computer is compromised, an attacker won't get your private keys.

However, you do need to use *a* computer to generate your keys; you'll also need to load your keys on a computer whenever you certify other people's keys or change subkeys or UIDs; and you might need to use a computer to recover if something goes wrong. For these higher-stakes operations, it's worthwhile to use a computer which is as secure as possible, even if that means that your more-secure computer is a little harder to use.

Securing a computer is difficult, *really* difficult. The more secure you want a computer to be, the more work it'll be to set that up, and -- probably -- the fewer features that computer will have, and the harder it'll be to use. The amount of work you want to spend securing a computer depends on who you think might be trying to get at your keys, and how capable, organized, and powerful they are.

[Threat Modeling](#threat-modeling) is the process of thinking about what attacks you might expect and how much work you want to spend mitigating it. Once you've thought about your threat model, [Securing a Computer](#securing-a-computer) talks about some approaches you can take to make a computer harder to compromise. From now on, this guide just talks about your "*secure computer*", whatever that means to you.

### Getting Started

Boot up your *secure computer*. Turn off any network connections, unplug any non-vital peripherals and check that nobody is watching over your shoulder: it's time to make some keys.

**A note on entropy.** Really good random numbers are needed to generate keys securely. We're not talking *kinda unpredictable*, we're talking about all-natural, organic, shade-grown, entropy. If you don't have good entropy, someone else might be able to guess your key, and that's be *really bad*. If you have one, now is the time to set up your hardware random number generator. If not, you probably don't need to worry: key generation will just take a little longer while your computer gathers entropy. If you're using a virtual machine, or another prefab environment you should definitely look at [What's Random?](#whats-random).

Okay, grab yourself your favorite terminal. Let's get started. 

## Master Key Generation

### Key Type and Capabilities

`amnesia@tails:~$ gpg --expert --gen-key`

The `expert` flag tells GPG that you might want to do something complex, and gives you more options (some of which you can get wrong). Don't worry though: you're following this guide, so you're an expert. The `gen-key` command tells GPG that we want to generate a new set of keys. GPG should reply with something like this:

~~~~~
gpg: WARNING: unsafe permissions on homedir `/tmp/gpg-guide-test/'
gpg (GnuPG) 1.4.12; Copyright (C) 2012 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection?
~~~~~

That's a lot of options. Those last two are only there because we're in `expert` mode. There are two things that GPG is asking right now:

1. What asymmetric-key algorithm do yo want to use? Your options are *RSA*, *DSA*, or *Elgamal*.
2. What *capabilities* do you want your key to have? Wait, what's a capability?

Capabilities are *things that a key can do*. GPG thinks of four capabilities, four different cryptographic applications of the same type of key:

* **Certify**. This is where you "sign" someone else's key: promising that you know that a key really belongs to the person it says it does.
* **Sign**. This is where you make an unforgeable "signature" on a file, document, or so on. Anyone can verify that only you could have signed it.
* **Encrypt**. Other people can encrypt messages to this key, and only you can decrypt them. Should probably be called "decrypt" instead.
* **Authenticate**. You can use this key to prove that you are who you claim. particularly useful when logging on to remote servers.

For various reasons related to the perceived strength of the ciphers, and the ways that they break when they fail, we're going to pick RSA keys. All RSA, all the time.

We're going to define our capabilities manually for the master key and some subkeys, so let's pick the last option.

`Your selection? 8`

GPG will now ask you to pick capabilities for your master key, asking something like this:

~~~~~
Possible actions for a RSA key: Sign Certify Encrypt Authenticate 
Current allowed actions: Sign Certify Encrypt 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? 
~~~~~

We're going to keep our master key in cold storage most of the time, probably on a USB drive or two stored in safe places. Our master key will be the "glue" that sticks our other keys together, binds them to our identity/ies, but won't be used for everyday tasks. Since it's going to be in cold storage, we shouldn't pick any of the capabilities that we use regularly. GPG has guessed that we want the `Sign Certify Encrypt` set of capabilities, but signing and encryption/decryption are everyday operations, so we should de-select them:

~~~~~
Your selection? S

Possible actions for a RSA key: Sign Certify Encrypt Authenticate 
Current allowed actions: Certify Encrypt 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? E

Possible actions for a RSA key: Sign Certify Encrypt Authenticate 
Current allowed actions: Certify 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? 
~~~~~

Now this prospective master key only has the `Certify` capability. Isn't certification a "day-to-day" activity? Perhaps you're the sort of person who wants to go around certifying every key you can, spreading the web-of-trust love to every corner of the globe? Sadly the OpenPGP standard and GPG in particular requires that a master key has the `Certify` capability, and simply doesn't allow subkeys to have that capability. There's no way around this, so we have to leave the `Certify` capability in place, and we'll have to get our master key out of cold storage and boot up a *secure computer* every time we want to certify someone's key. Such is life, let's shed a single solitary tear and move on.

`Your selection? Q`


### Key Size

Now GPG will ask you how big a key you would like:

~~~~~
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
~~~~~

GPG's suggested default is 2048 bits. With crypto keys, bigger is always more secure. Pick the biggest number you can think of that's 4096 or less:

`What keysize do you want? (2048) 4096`


### Expiration Time

We need to decide when this key should expire.

~~~~~
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 
~~~~~

"Author," -- you might be thinking -- "this is a pretty involved process, and I don't want to do it very often. Why would I want my key to expire." Good question, hypothetical straw-person reader. You're quite right: it really would be better if this key never expires. But even as we hope for the best, we should prepare for the worst.

Imagine some hypothetical future you. You use your key all the time, and loads of folks rely on it. Then: disaster strikes! You are attacked by a ninja clan, who (mistakenly?) think that you're a pirate sympathizer. They destroy all your computers, all your backups, everything. For good measure, they thwack you on the head and make you forget all your passwords and other secret plans. Now you have a problem: you've lost you key, but it's still valid. Other people might even send you encrypted messages or expect you to sign or authenticate things with it. Even if you create a new key, there's nothing to stop some poor confused soul from using your old, destroyed key by accident. Oh dear.

We're going to use expiration in an attempt to stave off this dire scenario, sort of like a dead-person's switch. When you set an expiry date on you key, in the event of scenario NINJA OBLIVION OMEGA, eventually the lost key will expire and people will stop trying to use it. However, in the RAINBOWS BUTTERFLIES UNICORNS situation, you're fine. If you still control the key, then you can always postpone the expiry date, even if it's already passed. Eerie, perhaps but useful.

With all that in mind, pick a time. The longer the time is, the longer someone might accidentally use a lost key. The shorter the time, the more frequently you'll have to pull your master key out of cold storage and boot up your *secure computer*. Remember your [threat model](#threat-modeling)? Let it guide you.

**Keyserverless usage note.** If you don't want to use keyservers, remember that you still have to distribute your key after you change its expiry date. If that's going to be especially difficult for you, consider that when picking a time.

For most folks, a year is a perfectly reasonable period.

~~~~~
Key is valid for? (0) 1y
Key expires at 06 Apr 2064 07:22:28 PM CST
Is this correct? (y/N) y
~~~~~


### User IDs

Now you need to tell GPG who you think you are:

~~~~~
You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

Real name: 
~~~~~

Oh my. Gentle reader, you may ask: "What is this madness?! Why does GPG want to know who I am; is it part of *The System*?". This is probably a good time to talk about identities, and what on earth they mean. This is just an intro, for more detail, take a look at [Nyms for UIDs in the Web of Trust](#nyms).

When you use GPG, you're probably trying to communicate with *people*. Fundamentally, you don't really care whether you're encrypting to a particular key. What you want is to know that you're encrypting to a particular *person*'s key. You want that person to be able to read the message, and nobody else. This is what user IDs (UIDs) are all about. A UID is an identifier which indicates a particular person by name and address. In theory, an address could be any way to reach someone, like "subspace channel 23571, towards relay station twelve". In practice, OpenPGP and GPG only understand email addresses and things that look like email addresses.

Think of the name that everyone knows you by, and the address that you tell people when you actually want them to email you. Those are probably the best options to put here. You probably don't need to worry about about a comment. Some people use a comment to distinguish personal and work email accounts. In practice, you'll probably be fine without it. Again, [nyms](#nyms) are a complicated topic.

~~~~~
Real name: Ada Lovelace
Email address: ada@enchantressofnumbers.net
Comment: 
You selected this USER-ID:
    "Ada Lovelace <ada@enchantressofnumbers.net>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
~~~~~


### Passphrase

~~~~~
You need a Passphrase to protect your secret key.

Enter passphrase:
~~~~~

Your passphrase is used to encrypt your key when it's stored on disk. This prevents someone who gets access to your secret keyring from using your key. You want to pick a passphrase so incredibly complex that a nobody will ever guess it, even if they use a computer to try guessing words and combinations of words, and so on. You also want a passphrase so simple and memorable that you'll never forget it. That sounds pretty tough. This isn't a guide to passphrases. What's your [threat model](#threat-model) again? GPG won't echo your passphrase back as you type it.


### Wait for Entropy

Now that you've filled out GPG's standard application form (in triplicate), your computer is actually going to generate some keys. About time.

~~~~~
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
~~~~~

What is this? Well, this could take a little while. Perhaps go get some tea?

~~~~~
Not enough random bytes available.  Please do some other work to give
the OS a chance to collect more entropy! (Need 281 more bytes)
...+++++

Not enough random bytes available.  Please do some other work to give
the OS a chance to collect more entropy! (Need 195 more bytes)
~~~~~

Ugh, a few more minutes. If we had a random number generator, this would be a lot easier.

~~~~~
...+++++
gpg: /tmp/gpg-guide-test//trustdb.gpg: trustdb created
gpg: key CDCD72AF marked as ultimately trusted
public and secret key created and signed.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
gpg: next trustdb check due at 2063-04-06
pub   4096R/CDCD72AF 2063-04-06 [expires: 2064-04-06]
      Key fingerprint = DDC6 93BF 8FC1 3036 36D2  CCFB 4771 324A CDCD 72AF
uid                  Ada Lovelace <ada@enchantressofnumbers.net>
~~~~~

Okay, great! We have a master key. You can drink that cup of tea now, you've earned it! Maybe even a biscuit too.


## Modifying a Key

### The Key Editor

We have a master key with the `Certify` capability, now it's time to to generate some subkeys which can `Sign`, `Encrypt` (really decrypt), and `Authenticate`, then polish things up and make any other tweaks this key needs.

`amnesia@tails:~$ gpg --expert --edit-key CDCD72AF`

That string of eight letters and numbers is the *key ID*, a short way to refer to this master key, all it's UIDs, signatures, any subkeys, or other stuff it might pick up over time. It's the last eight character of the key's fingerprint, which GPG just told you when it completed generating your key.

~~~~~
gpg (GnuPG) 1.4.12; Copyright (C) 2012 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

pub  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06  usage: C   
                     trust: ultimate      validity: ultimate
[ultimate] (1). Ada Lovelace <ada@enchantressofnumbers.net>

gpg> 
~~~~~

This is GPG's key-editing setup, and there are lots of things you can do from here. There are loads of commands, and you can see them all if you ask for some `help`. Let's walk through what you're being shown here then move on to make some subkeys.

At the top, there's a copyright notice and disclaimer; how boring. When we set up our everyday computer later, we can flick a switch to turn that off.

"Secret key is available." means exactly what it says on the can. It's possible to use the `edit-key` command when you *don't* have the secret key available, but anything that requires the secret key won't work, obviously.

The next line tells us that this is a 4096-bit RSA key, and that the key ID (last eight characters of the fingerprint) are `CDCD72AF`. Your key ID (and fingerprint) will be different. The creation time and expiry time should just be today's date and that plus a bit (a year, in this case). Ada is a well-known time-traveler. Usage tells us that this key only has the `Certify` capability.

Trust and validity are concepts related to the web of trust, and GPG's work to determine if a key *really belongs* to the person it says it does. Since you just made this key, GPG has assigned the highest possible values to it.

The next line is the key's UID. There's just one, but it has a little `(1)` beside it anyway, just in case some others show up.

### Additional UIDs

There are some other changes or tweaks that require the master key's signature. We should make those now.

Perhaps you want to include a work email address as another UID:

~~~~~
gpg> adduid
Real name: Ada Lovelace
Email address: ada@analyticalengine.com
Comment: 
You selected this USER-ID:
    "Ada Lovelace <ada@analyticalengine.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
~~~~~

You could also add a photo:

~~~~~
gpg> addphoto

Pick an image to use for your photo ID.  The image must be a JPEG file.
Remember that the image is stored within your public key.  If you use a
very large picture, your key will become very large as well!
Keeping the image close to 240x288 is a good size to use.

Enter JPEG filename for photo ID: /home/ada/.face.jpg
This JPEG is really large (27348 bytes) !
Are you sure you want to use it? (y/N) y
Is this photo correct (y/N/q)? y
~~~~~

After you've made the changes you want, you should be left with something like this:

~~~~~
pub  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06  usage: C   
                     trust: ultimate      validity: ultimate
[ultimate] (1)  Ada Lovelace <ada@enchantressofnumbers.net>
[ unknown] (2). Ada Lovelace <ada@analyticalengine.com>
[ unknown] (3)  [jpeg image of size 27348]
~~~~~


## Subkey Generation

Now it's time to make some subkeys. These are the keys you'll be using every day. This guide is supposed to help you use a smartcard to store your everyday keys so that you don't actually need to tell your everyday computer what their secret parts are. If you don't want or have a smartcard, some of these steps will be slightly different.


### Subkey Type, Capabilities, & Size

`gpg> addkey`

Once you type in your passprase, GPG gives us the full list of options since we're in `expert` mode again.

~~~~~
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection? 
~~~~~

We're all RSA all the time. We're going to assign capabilities manually, so pick the final option just like last time.

`Your selection? 8`

We're going to use a different key for each capability. GPG defaults to `Sign Encrypt` for RSA subkeys:

~~~~~
Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: Sign Encrypt 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? 
~~~~~

Disable the `Encrypt` capability, and continue:

~~~~~
Your selection? E

Possible actions for a RSA key: Sign Encrypt Authenticate 
Current allowed actions: Sign 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q
~~~~~

Again, we need to pick the size for this key:

~~~~~
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
~~~~~

If you're not going to use a smartcard right now, pick a really big number that's 4096 or less. If you are going to use a smartcard, you should pick 3072, since that's the largest key that a smartcard can deal with. There's no very good reason for that particular limit, but that's definitely the limit.

If you're using a smartcard, you may be wondering why we're going to generate this key on a computer rather than on the smartcard itself. If we generate the key on a computer first, then we can keep a backup with our master key. If your smart card later gets left on a bus or run over by one, you have a backup copy of the keys, so you can copy them to a new smart card, and continue to decrypt old messages.


### Subkey Expiry

~~~~~
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 
~~~~~

Unlike a master key which is rather a hassle to replace, you can rotate subkeys relatively regularly by generating new ones and revoking the old ones. Other folks who update your key from a keyserver should be able to switch over to the new one without even noticing that they did. This is more difficult if you choose not to use keyservers. 

If you're using a smartcard, you might not want your subkeys to expire: you can just keep using them until something goes wrong, and generate new ones then. If you're not using a smartcard, you might want to pick a reasonable expiry period. You have to bring your master key out of cold storage and boot up your *secure computer* to change subkeys, so you might want to make it a large fraction of your master key expiry period, and take that opportunity to rotate subkeys and postpone your master key's expiration at the same time.

~~~~~
Key is valid for? (0) 
Key does not expire at all
Is this correct? (y/N) y
Really create? (y/N) y
~~~~~

While you're waiting for GPG to gather entropy for this subkey, perhaps you might want to browse <http://entropykey.co.uk>. A UK company called Simtek Electronics makes a USB device which they claim generates truly random quantum entropy. Both the firmware and hardware are closed, so it's hard to verify that claim. Still, the open source MIT-license software is available from the site, and included in Debian and Ubuntu.

### Repeat

Now we've got a subkey that can `Sign`. Repeat these steps two more times to generate an `Encrypt` and an `Authenticate` subkey, and you should get something that looks like this.

~~~~~
pub  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06  usage: C   
                     trust: ultimate      validity: ultimate
sub  3072R/FACBBB45  created: 2063-04-06  expires: never       usage: S   
sub  3072R/8FE97F11  created: 2063-04-06  expires: never       usage: E   
sub  3072R/94C86525  created: 2063-04-06  expires: never       usage: A   
[ultimate] (1). Ada Lovelace <ada@enchantressofnumbers.net>

gpg>
~~~~~


### Finish Up

We have our keys all set. They're the right lengths, there are the right number of them, they do the right things, they have the right UIDs. Everything looks great.

~~~~~
pub  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06  usage: C   
                     trust: ultimate      validity: ultimate
sub  3072R/FACBBB45  created: 2063-04-06  expires: never       usage: S   
sub  3072R/8FE97F11  created: 2063-04-06  expires: never       usage: E   
sub  3072R/94C86525  created: 2063-04-06  expires: never       usage: A   
[ultimate] (1)  Ada Lovelace <ada@enchantressofnumbers.net>
[ unknown] (2). Ada Lovelace <ada@analyticalengine.com>
[ unknown] (3)  [jpeg image of size 27348]
~~~~~

If you're happy with the way things look, you can save your changes and quit:

`gpg> save`

If you quit without saving, all your changes in this session will be discarded.


## Primary Export & Backup

Now's the time to copy this key somewhere.

`amnesia@tails:~$ gpg --armor --output /media/cold-storage/ada-master.gpg --export-secret-keys CDCD72AF`

The `armor` signal tells GPG to write an "ASCII-armored" text file, rather than a binary one. The `output` parameter specifies the filename to save the keys to. The `export-secret-keys` command does just what it sounds like. The regular `export` command only exports public keys, for your safety and convenience. Note that the GPG command always comes last, after all its flags and parameters. Just for safe keeping, let's make some backups.

~~~~~
amnesia@tails:~$ gpg --armor --output /media/key-backup-alpha/ada-master.gpg --export-secret-keys CDCD72AF
amnesia@tails:~$ gpg --armor --output /media/key-backup-omega/ada-master.gpg --export-secret-keys CDCD72AF
~~~~~

If you want to make a paper backup, you could just print one of those "ASCII-armored" export files. If you ever have to use it, you'll have a fun job typing all that back in. There's probably a good way to use a 2D barcode. Paper backups are left as an exercise to the reader.

Note also that key is only as secure as the least safe place that you have it backed up. If you export it to an unencrypted removable drive and someone else gets their hands on that drive, only the complexity of your passphrase is standing between them and total control of your key.

Now that the keys are safely exported and backed up, we're going to mutilate them a little so that the version copied to your everyday computer is as innocuous as possible.


## Smartcards

This next section is all about smartcards. If you don't want to use a right now smartcard, you can skip it. Don't worry, you can always move to a smartcard some other time.

There are [more thoughts](#smartcard-selection) on picking a smartcard and the advantages and drawbacks later. For now, this guide assumes that you've picked a smartcard and reader, and have them ready to go. The GPG manual has [some instructions](http://www.gnupg.org/howtos/card-howto/en/smartcard-howto-single.html) for setting up your computer to use a smartcard and reader.


### Smartcard Setup

First, let's take a look at your smartcard, and check that it works. 

`amnesia@tails:~$ gpg --card-edit`

You should see something like this:

~~~~~
gpg: detected reader `Generic CCID Reader 00 00'
Application ID ...: D2760001240101010001000000490000
Version ..........: 2.0
Manufacturer .....: ZeitControl
Serial number ....: 00000101
Name of cardholder: [not set]
Language prefs ...: de
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Private DO 1 .....: [not set]
Private DO 2 .....: [not set]
Signature PIN ....: forced
Max. PIN lengths .: 32 32 32
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [not set]
Encryption key....: [not set]
Authentication key: [not set]
General key info..: [none]
~~~~~

If you don't, make sure that GPG is set up to use your smartcard. Perhaps check the [GPG manual pages on the topic](http://www.gnupg.org/howtos/card-howto/en/smartcard-howto-single.html). If you find another guide which is better for your smart card, please send it over.

If GPG can see your card, you can set it up. Let's turn on admin mode so that we can make changes.

~~~~~
gpg/card> admin
Admin commands are allowed
~~~~~


### Authorization PINs

The default user PIN is `123456` and the default admin PIN is `12345678`. Let's start by changing these.

~~~~~
gpg/card> passwd
gpg: OpenPGP card no. D276000124010200000500001A640000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 
~~~~~

The admin PIN is used to make changes to the card, while the user PIN actually lets to tell the card to decrypt or sign things using the onboard keys. If you (or some malicious outsider) enters the user PIN wrong three times, the card is locked, and the Admin PIN is needed to unlock it. If you use a GSM cellphone, this is somewhat like your SIM card's PIN and PUK codes. Entering an incorrect admin PIN three times destroys the card. You probably want to avoid that.

The admin PIN isn't used every day, and you only need it to reconfigure the card or fix it if you mess up your PIN. It doesn't need to be something you can remember, just find when all else fails. It's probably a good idea to use the longest admin PIN your card accepts (in this example, 32 digits long). You should probably write the admin PIN down a text file on your master key cold storage and backup disks.

~~~~~
Your selection? 3
gpg: 3 Admin PIN attempts remaining before card is permanently locked

Please enter the Admin PIN
                 
New Admin PIN
                     
New Admin PIN
PIN changed.     
~~~~~

Now change the user PIN. This is the PIN that you'll enter every time you use the card. Pick something short and memorable enough that you'll be able to enter it frequently without difficulty.

~~~~~
Your selection? 1

Please enter the PIN
           
New PIN
               
New PIN
PIN changed.     
~~~~~

All done here.

~~~~~
Your selection? Q

gpg/card> 
~~~~~


### Personal Attributes

You can also set some personal attributes. Many of them do very little in practice, though `login` has a use.

~~~~~
gpg/card> name
Cardholder's surname: Lovelace
Cardholder's given name: Ada
gpg: 3 Admin PIN attempts remaining before card is permanently locked

Please enter the Admin PIN

gpg/card> lang
Language preferences: en

gpg/card> sex
Sex ((M)ale, (F)emale or space): F
~~~~~

The `login` attribute specifies your typical username on servers where you might authenticate yourself using a key. The `url` attribute specifies a place where someone could find your public key online. There's no need to put it there right now, but you can pick the URL with the intent to put it there later.

~~~~~
gpg/card> login
Login data (account name): ada

gpg/card> url
URL to retrieve public key: https://enchantressofnumbers.net/key.asc
~~~~~

You don't need to set these, but if you do, you might end up with something a little like this.

~~~~~
gpg: detected reader `Generic CCID Reader 00 00'
Application ID ...: D2760001240101010001000000490000
Version ..........: 2.0
Manufacturer .....: ZeitControl
Serial number ....: 00000101
Name of cardholder: Ada Lovelace
Language prefs ...: en
Sex ..............: unspecified
URL of public key : https://enchantressofnumbers.net/key.asc
Login data .......: ada
Private DO 1 .....: [not set]
Private DO 2 .....: [not set]
Signature PIN ....: forced
Max. PIN lengths .: 32 32 32
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [not set]
Encryption key....: [not set]
Authentication key: [not set]
General key info..: [none]
~~~~~

When you're done: 

`gpg/card> quit`

### Smartcard Export

Now that your smartcard is ready for use, we can move the keys onto it. When you move a key to a smartcard, GPG deletes that private key from your keyring, and replaces it with a a stub noting that the key is actually on a particular smartcard. That's how GPG knows to prompt you for a card rather than just assuming that the key is unusable without its private parts.

~~~~~
amnesia@tails:~$ gpg --edit-key CDCD72AF
gpg: WARNING: unsafe permissions on homedir `/tmp/gpg-guide-test/'
gpg (GnuPG) 1.4.12; Copyright (C) 2012 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
gpg: next trustdb check due at 2014-08-01
pub  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06  usage: C   
                     trust: ultimate      validity: ultimate
sub  3072R/FACBBB45  created: 2063-04-06  expires: never       usage: S   
sub  3072R/8FE97F11  created: 2063-04-06  expires: never       usage: E   
sub  3072R/94C86525  created: 2063-04-06  expires: never       usage: A   
[ultimate] (1)  Ada Lovelace <ada@enchantressofnumbers.net>
[ unknown] (2). Ada Lovelace <ada@analyticalengine.com>
[ unknown] (3)  [jpeg image of size 27348]
~~~~~

We need to export keys to a card one at a time. Let's start with the signing key. First we need to toggle to the secret key listing.

~~~~~
gpg> toggle

sec  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06
ssb  3072R/FACBBB45  created: 2063-04-06  expires: never     
ssb  3072R/8FE97F11  created: 2063-04-06  expires: never     
ssb  3072R/94C86525  created: 2063-04-06  expires: never     
(1)  Ada Lovelace <ada@enchantressofnumbers.net>
(2)  Ada Lovelace <ada@analyticalengine.com>
(3)  [jpeg image of size 27348]
~~~~~


Now select the signing subkey. The key editor uses `key #` to select a subkey. Unlike UIDs, subkeys don't have helpful number hints by them, so you just need to count. How hard could it be?

~~~~~
gpg> key 1

sec  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06
ssb* 3072R/FACBBB45  created: 2063-04-06  expires: never     
ssb  3072R/8FE97F11  created: 2063-04-06  expires: never     
ssb  3072R/94C86525  created: 2063-04-06  expires: never     
(1)  Ada Lovelace <ada@enchantressofnumbers.net>
(2)  Ada Lovelace <ada@analyticalengine.com>
(3)  [jpeg image of size 27348]
~~~~~

Notice how the signing subkey key now has a little star beside it? That's means it's selected.

~~~~~
gpg> keytocard
gpg: detected reader `Generic CCID Reader 00 00'
Signature key ....: [not set]
Encryption key....: [not set]
Authentication key: [not set]

Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1
~~~~~

Now repeat this for the other two subkeys. Make sure to put them in the right slots on the card. When you're done, the secret key view should look like this.

~~~~~
sec  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06
ssb  3072R/FACBBB45  created: 2063-04-06  expires: never     
                     card-no: 0005 00000101
ssb  3072R/8FE97F11  created: 2063-04-06  expires: never     
                     card-no: 0005 00000101
ssb  3072R/94C86525  created: 2063-04-06  expires: never     
                     card-no: 0005 00000101
(1)  Ada Lovelace <ada@enchantressofnumbers.net>
(2)  Ada Lovelace <ada@analyticalengine.com>
(3)  [jpeg image of size 27348]
~~~~~

The smartcard is ready to go. Save your changes and quit.

`gpg> save`


## Everyday Export

Now we can export the subkeys for use on our everyday machine. We're going to use GPG's `export-secret-subkeys` key command, which outputs a file containing the public part of the master key, the public and private parts of the subkey, and all the metadata, like UIDs, expiry times, and so on.

If you're using a smartcard, this output will actually contain *no* secret keys at all! It won't contain the secret part of the master key, because we're not exporting that anyway, and it won't contain the secret parts of the subkeys, because those have been replaced by pointers noting that the secret part is on a smartcard.

`amnesia@tails:~$ gpg --armor --output /media/everyday-usb/ada-everyday.gpg --export-secret-keys CDCD72AF`

This is the file that you'll import on your everyday machine.

## Finishing

We're done with key generation now. If you've been following through all the steps, you probably have four USB thumb drives and one smartcard/reader.

* Cold storage for your master key and all your subkeys. Put this on your home safe or your piggy bank, or wherever you keep things safely at home.
* Two backups of your cold storage. Take these and put them somewhere safe. Give one to a friend who lives far away. Put another somewhere safe outside your home.
* A thumb drive destined for your everyday computer, with public parts of your keys.
* A smartcard containing the public and private parts of all of your subkeys.

If you've got all that, you're ready to go. You can shut down your *secure computer* now. Next, we'll walk through setting up your everyday computer.


# Everyday Setup & Usage
## Key Import
## `gpg.conf`
## Daily Usage

# Web of Trust
## Trust models
## Key-signing
## Certification Policy
## Key servers

# Transitions, Catastrophes, and Disasters


# Background

In creating a secure system and secure processes, it is important to know what one’s goals are. This section is not an introduction to security design, but hopefully an explanation of the goals and approach that the author used to plan this system and write this guide.


# Extra Topics
## Securing a Computer {#securing-a-computer}
## Smartcard Selection {#smartcard-selection}
## What's Random? {#whats-random}
## Threat Modeling {#threat-modeling}
## Key-server Risks
## Nyms for UIDs in the Web of Trust {#nyms}
## GPG vs. GPG2

