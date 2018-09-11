Introduction
============

This guide is intended to enable an experienced technical user to securely generate GPG keys, back them up, and put them on smartcards.

The version of GPG used in this guide is `1.4.12-7` from Debian's wheezy/main repository.


Caveats
-------

As with many things, this guide has limitations.

The author is a Debian user, so all the specific instructions are based on Debian 7.x. If you use a different version of Debian or Debian-like OS such as Ubuntu or Mint, things might just work for you unchanged. If you’re an OSX user you might need to change something here or there. Unfortunately, the author is not able to provide advice for Windows.

This guide is targeted at a technically experienced user. It focuses on use of the terminal, not on graphical applications. Sadly, this makes it inaccessible to a lot of people. At the time of writing, GPG is not a very user-friendly piece of software, and there aren't many graphical frontends which both work well and support the advanced usage scenario described in this guide. If you find an application which works and is compatible with this guide, please get in touch: the author would love to hear from you!

If you try this guide and it doesn’t work for you, please email the author and tell them what broke! If you use OSX or Windows, you are invited to port this guide for your OS, and to let the author know what the right instructions are for you.


License
-------

The license for this work is [Creative Commons Attributuion-ShareAlike-3.0 USA](
https://creativecommons.org/licenses/by-sa/3.0/us). You are welcome to re-use it under that license, but the author requests that you send changes upstream by email or pull request rather than forking outright.


Target Setup
------------

This is a guide for setting up a master key with three specific subkeys, backed up, and stored on smartcards. If you follow this guide through, you should end up with:

* backups of your GPG master key & subkeys on removable storage,
* a smartcard with your GPG master key, to use when certifying other people's keys,
* a smartcard with your everyday keys,
* your main computer set up and ready to use GPG, but without any private keys,
* revocation certificates stored in a safe place.


Before you Start
================

Materials
---------

You may want to get some things ready before following this guide. To follow all the steps, you'll need:

1. Your main computer: the one that you use everyday and want to use GPG on.
2. A "secure" computer, whatever that means.
3. Two smartcard(s) and readers for them.
4. Some storage for your master key and backups: a handful of flash drive(s), and perhaps a printer.
5. A hardware random number generator, if you have one.


Environment/Preparation
-----------------------

Cryptography is only a secure as the computer that's running it. If your private keys are stored on a computer which is compromised, the attacker can decrypt your messages, make valid signatures, or impersonate you. One of the goals of the setup described here is that you never end up showing your private keys to your everyday computer. Even if your everyday computer is compromised, an attacker won't get your private keys.

However, you do need to use *a* computer to generate your keys and you might need to use a computer to recover if something goes wrong. For these higher-stakes operations, it's worthwhile to use a computer which is as secure as possible, even if that means that your more-secure computer is a little harder to use.

Securing a computer is *really* difficult. The more secure you want a computer to be, the more work it'll be to set that up, and -- probably -- the fewer features that computer will have and the harder it'll be to use. The amount of work you want to spend securing a computer depends on who you think might be trying to get at your keys, and how capable, organized, and powerful they are.

Threat modeling is the process of thinking about what attacks you might expect and how much work you want to spend mitigating those attacks. Once you've thought about your threat model, you can pick the right secure computer for you. From now on, this guide just talks about your "*secure computer*", whatever that means to you.

Getting Started
---------------

Boot up your *secure computer*. Turn off any network connections, unplug any non-vital peripherals and check that nobody is watching over your shoulder: it's time to make some keys.

**A note on entropy.** Really good random numbers are needed to generate keys securely. We're not talking *kinda unpredictable*, we're talking about all-natural, organic, shade-grown, entropy. If you don't have good entropy, someone else might be able to guess your key, and that'd be *really bad*. If you have one, now is the time to set up your hardware random number generator. If not, you probably don't need to worry: key generation will just take a little longer while your computer gathers entropy. If you're using a virtual machine, or another prefab environment you should definitely worry about this

Okay, grab yourself your favorite terminal. Let's get started. 


Master Key Generation
=====================

Initial Configuration
---------------------

Some of the options in GPG's config file influence the parameters used when generating keys. Edit your `~/.gnupg/gpg.conf` to add the following lines. Make sure to remove any similar directives which are already there. If you like, back up your current config file and decide which parameters to choose later when we discuss [`gpg.conf`](#config) in more depth. For now, you can safely just cargo cult this:

~~~~
no-greeting
personal-digest-preferences SHA512
personal-cipher-preferences AES256 AES
cert-digest-algo SHA512
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed
s2k-cipher-algo AES256
s2k-digest-algo SHA512
s2k-mode 3
s2k-count 65011712
~~~~

Key Type and Capabilities
-------------------------

`amnesia@amnesia:~$ gpg --expert --gen-key`

The `expert` flag tells GPG that you might want to do something complex, and gives you more options (some of which you can get wrong). Don't worry though: you're following this guide, so you're an expert. The `gen-key` command tells GPG that we want to generate a new set of keys. GPG should reply with something like this:

~~~~~
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

We're going to keep our master key in cold storage most of the time, probably on a USB drive or two stored in safe places. Our master key will be the "glue" that sticks our other keys together, binds them to our identity/ies, but won't be used for everyday tasks. It's always a good idea to pick the minimal set of capabilities.

Sadly the OpenPGP standard and GPG in particular requires that a master key has the `Certify` capability, and simply doesn't allow subkeys to have that capability. There's no way around this, so we have to leave the `Certify` capability in place, and we'll use this master key every time we want to certify other people's keys. Many smartcards won't accept a key which doesn't have one of the sign/authenticate/encrypt capabilities. Since we want to put the master key on a smartcard, let's leave the `Sign` capability too.

~~~~~
Your selection? E

Possible actions for a RSA key: Sign Certify Encrypt Authenticate 
Current allowed actions: Sign Certify

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q
~~~~~

Key Size
--------

Now GPG will ask you how big a key you would like:

~~~~~
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048)
~~~~~

GPG's suggested default is 2048 bits. With crypto keys, bigger is always more secure, so your gut reaction should be to pick 4096 bits. Unfortunately, may smartcards don't support keys larger than 3072 bits, so let's pick that size. There's no very good reason for that particular limit, but that's definitely what it is.

You may be wondering why we're going to all this trouble to generate this key on a computer rather than on the smartcard itself. If we generate the key on a computer first, then we can keep a backup on removable media. If your smart card later gets left on a bus or run over by one, you have a backup. You can copy it to a new smartcard and pick up where you left off.

`What keysize do you want? (2048) 3072`


Expiration Date
---------------

We need to decide when this key should expire.

~~~~~
Requested keysize is 3072 bits
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

With all that in mind, pick a time. The longer the time is, the longer someone might accidentally use a lost key. The shorter the time, the more frequently you'll have to get out your master-key smartcard and and push back the clock. Remember your threat model? Let it guide you.

**Keyserverless usage note.** If you don't want to use keyservers, remember that you still have to distribute your key after you change its expiry date. If that's going to be especially difficult for you, consider that when picking a time.

For most folks, a six months is a perfectly reasonable period.

~~~~~
Key is valid for? (0) 6m
Key expires at 06 Apr 2064 07:22:28 PM CST
Is this correct? (y/N) y
~~~~~


User IDs
--------

Now you need to tell GPG who you think you are:

~~~~~
You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

Real name: 
~~~~~

Oh my. Gentle reader, you may ask: "What is this madness?! Why does GPG want to know who I am; is it part of *The System*?". This is probably a good time to talk about identities, and what on earth they mean.

When you use GPG, you're probably trying to communicate with *people*. Fundamentally, you don't really care whether you're encrypting to a particular key. What you want is to know that you're encrypting to a particular *person*'s key. You want that person to be able to read the message, and nobody else. This is what user IDs (UIDs) are all about. A UID is an identifier which indicates a particular person by name and address. In theory, an address could be any way to reach someone, like "subspace channel 23571, towards relay station twelve". In practice, OpenPGP and GPG only understand email addresses and things that look like email addresses.

Think of the name that everyone knows you by, and the address that you tell people when you actually want them to email you. Those are probably the best options to put here. Don't enter a comment, you will only ever regret it. Again, nyms are a complicated topic.

~~~~~
Real name: Ada Lovelace
Email address: ada@enchantressofnumbers.net
Comment: 
You selected this USER-ID:
    "Ada Lovelace <ada@enchantressofnumbers.net>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
~~~~~


Passphrase
----------

~~~~~
You need a Passphrase to protect your secret key.

Enter passphrase:
~~~~~

Your passphrase is used to encrypt your key when it's stored on disk. This prevents someone who gets access to your secret keyring from using your key. You want to pick a passphrase so incredibly complex that a nobody will ever guess it, even if they use a computer to try guessing words and combinations of words, and so on. You also want a passphrase so simple and memorable that you'll never forget it. That sounds pretty tough. This isn't a guide to passphrases. What's your threat model again? GPG won't echo your passphrase back as you type it.


Wait for Entropy
----------------

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
pub   3072R/CDCD72AF 2063-04-06 [expires: 2064-10-06]
      Key fingerprint = DDC6 93BF 8FC1 3036 36D2  CCFB 4771 324A CDCD 72AF
uid                  Ada Lovelace <ada@enchantressofnumbers.net>

Note that this key cannot be used for encryption.  You may want to use
the command "--edit-key" to generate a subkey for this purpose.
~~~~~


Backup and back off
===================

Back up
-------

Now your master key is available. You want to export it up to some secure media and then never speak of it again. Every time a computer has access to your master key, that's a chance for everything to go wrong. We're going to export it to cold storage, and then test to make sure that we actually did that right.

~~~~
amnesia@amnesia:~$ gpg --armor --output /media/cold-storage/ada-master_private.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/cold-storage/ada-master_public.gpg --export CDCD72AF
~~~~

That string of eight letters and numbers at the end is the *key ID*, a short way to refer to this master key, all it's UIDs, signatures, any subkeys, or other stuff it might pick up over time. It's the last eight character of the key's fingerprint, which GPG just told you when it completed generating your key. Key IDs aren't securely unique, but that doesn't matter right now.

The `armor` signal tells GPG to write an "ASCII-armored" text file, rather than a binary one. The `output` parameter specifies the filename to save the keys to. The `export-secret-keys` command does just what it sounds like: it exports secret keys, but not the public parts. The regular `export` command only exports public keys, for your safety and convenience. Note that the GPG command always comes last, after all its flags and parameters. Just for safe keeping, let's make some backups.

~~~~~
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-alpha/ada-master_private.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-alpha/ada-master_public.gpg --export CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-omega/ada-master_private.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-omega/ada-master_public.gpg --export CDCD72AF
~~~~~

If you want to make a paper backup, you could print one of those "ASCII-armored" export files. If you ever have to use it, you'll have a fun job typing all that back in. There's probably a good way to use a 2D barcode. Paper backups are left as an exercise to the reader.

The key is only as secure as the least safe place that you have it backed up. If you export it to an unencrypted removable drive and someone else gets their hands on that drive, only the complexity of your passphrase is standing between them and total control of your key.

Back off
--------

Okay, now you've probably exported your key to some safe cold-storage. I say *probably* because let's just make sure, eh? Delete your secret keyring and make sure you can get your keys back from the backups you just made

`rm ~/.gnupg/secring.gpg`

If you're using something like TAILS, rebooting is a great idea, but remember to edit your `gpg.conf` again after you boot back up. Once you've rebooted, try to re-import the master key from that backup you just made.

~~~~
amnesia@amnesia:~$ gpg --import /media/cold-storage/ada-master_public.gpg
gpg: key CDCD72AF: public key "Ada Lovelace <ada@enchantressofnumbers.net>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
~~~~

~~~~
amnesia@amnesia:~$ gpg --import /media/cold-storage/ada-master_private.gpg
gpg: key CDCD72AF: secret key imported
gpg: key CDCD72AF: public key "Ada Lovelace <ada@enchantressofnumbers.net>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1
gpg:       secret keys read: 1
gpg:   secret keys imported: 1
~~~~

All that stuff about `secret keys imported`: that's GPG telling us that it managed to retrieve the files. To be on the safe side, delete your keyring/reboot, and test the other two backups just the same way.

Okay, great! We have a master key. You can drink that cup of tea now, you've earned it! Maybe even a biscuit too. While you're waiting for it to brew, go put your cold storage and your backups somewhere safe. If all goes well, you'll never need to touch them again.


Smartcards
==========

This next section is all about smartcards. Smartcards are designed to securely store cryptographic keys without ever revealing them. When you use a smartcard, all the crypto happens *on the card itself*. Your computer sends the smartcard a message like "Please decrypt this file." and the smartcard does the decryption and sends back the plaintext without ever revealing the secret key.

Unfortunately, most smartcards and readers are proprietary. That means that it's really hard to audit them to ensure that they're truly secure and don't have any backdoors. It's totally plausible that some smarcards might have bugs that allow timing attacks, and perhaps some even have backdoors which *will* allow them to divulge the secret key.

For safety's sake, you might want to *assume* that your smartcard has a backdoor or vulnerability and treat it like a (less-secure) USB thumb drive. On the bright side, your card might actually be secure and give you an extra layer of insualtion against attackers!

You can think long and hard about which smartcard you want, and who you trust to make them safely. For now, let's assume that you've picked a smartcard and reader, and have them ready to go. The GPG manual has [some instructions](http://www.gnupg.org/howtos/card-howto/en/smartcard-howto-single.html) for setting up your computer to use a smartcard and reader.


Smartcard Setup
---------------

First, let's take a look at your smartcard, and check that it works. 

`amnesia@amnesia:~$ gpg --card-edit`

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


Authorization PINs
------------------

Before setting up the PINs for your smartcard, it is important to have a safe place to store them. A password manager may help you generate, store _(and label)_ each of the three PINs (admin PIN, user PIN, and reset code) we'll be setting up. Some of the commands we'll be using may "time out" abruptly and force you to start over in entering a new PIN if you take slightly too long to type it, so it is recommended that you choose what these three PINs will be ahead of time. A password manager will help you do that.

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

The admin PIN is used to make changes to the card, while the user PIN actually lets the card decrypt or sign things using the onboard keys. If you (or some malicious outsider) enters the user PIN wrong three times, the card is locked, and the Admin PIN is needed to unlock it. If you use a GSM cellphone, this is somewhat like your SIM card's PIN and PUK codes. Entering an incorrect admin PIN three times destroys the card. You probably want to avoid that.

The admin PIN isn't used every day, and you only need it to reconfigure the card or fix it if you mess up your PIN. It doesn't need to be something you can remember, just find when all else fails. It's probably a good idea to use the longest admin PIN your card accepts (in this example, 32 digits long). You should probably write the admin PIN down a text file on your master key cold storage and backup disks.

~~~~~
Your selection? 3
gpg: 3 Admin PIN attempts remaining before card is permanently locked

Please enter the Admin PIN
                 
New Admin PIN
                     
New Admin PIN
PIN changed.     
~~~~~

Now change the user PIN. This is the PIN that you'll enter every time you use the card. This first card is just for your master key which you'll only use for certification. That doesn't happen very often, so feel free to pick a long user PIN, as long as you record it somewhere safe or don't forget it.

~~~~~
Your selection? 1

Please enter the PIN
           
New PIN
               
New PIN
PIN changed.     
~~~~~

You should also set a reset code. The reset code is useful in case you have forgotten both the admin and user PINs. It tells the card to completely reset itself, deleting all the keys, and returning to the factory configuration. Since you have backups of everything, this helps avoid turning your fancy smartcard into a useless piece of plastic.

~~~~~
Your selection? 4

Please enter the PIN
           
New Reset Code
               
New Reset Code
Reset Code set.     
~~~~~

All done here.

~~~~~
Your selection? Q

gpg/card> 
~~~~~


Personal Attributes
-------------------

You can also set some personal attributes. Many of them do very little in practice, though `login` can have a use.

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

The Key Editor
--------------

~~~~~
amnesia@amnesia:~$ gpg --edit-key CDCD72AF
Secret key is available.

pub  3072R/CDCD72AF  created: 2063-04-06  expires: 2063-04-06  usage: SC   
                     trust: unknown       validity: unknown
[ unknown] (1). Ada Lovelace <ada@enchantressofnumbers.net>
~~~~~

This is GPG's key-editing setup, and there are lots of things you can do from here. There are loads of commands, and you can see them all if you ask for some `help`. Let's walk through what you're being shown here then put your master key on a smartcard.

"Secret key is available." means exactly what it says on the tin. It's possible to use the `edit-key` command when you *don't* have the secret key available, but anything that requires the secret key won't work, obviously.

The next line tells us that this is a 3072-bit RSA key, and that the key ID (last eight characters of the fingerprint) are `CDCD72AF`. Your key ID (and fingerprint) will be different. The creation time and expiry time should just be today's date and that plus a bit (six months, in this case). Ada is a well-known time-traveler. Usage tells us that this key has the `Sign` and `Certify` capabilities.

Trust and validity are concepts related to the web of trust, and GPG's work to determine if a key *really belongs* to the person it says it does.

The next line is the key's UID. There's just one, but it has a little `(1)` beside it anyway, just in case some others show up.

Smartcard Export
----------------

Your smartcard is ready for use and it's time to move your master key over. When you move a key to a smartcard, GPG deletes that private key from your keyring, and replaces it with a a stub noting that the key is actually on a particular smartcard. That's how GPG knows to prompt you for a card rather than just assuming that the key is unusable without its private parts.

First we need to toggle to the secret key listing.

~~~~~
gpg> toggle

sec  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06
(1)  Ada Lovelace <ada@enchantressofnumbers.net>
~~~~~

Then we send it to the card. This can't be undone! Make sure that you've already made your backups, because there's no way to get the secret key back once you've sent it to the card. It sure would be a pity to go to all this trouble only to break the card in a freak balooning accident.

~~~~~
gpg> keytocard
Really move the primary key? (y/N) 
~~~~~

GPG is skeptical, but this is really what you want to do.

~~~~~
Really move the primary key? (y/N) y
gpg: detected reader `Generic CCID Reader 00 00'
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]

Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1
~~~~~

GPG will prompt you for your passphrase then the card's admin PIN. Do try to enter them correctly. If you do, you should see something like this:

~~~~~
sec  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06
                     card-no: 0005 00000101
(1)  Ada Lovelace <ada@enchantressofnumbers.net>
~~~~~

That little note with the card number means that GPG knows that the key is stores on the smartcard. Success! Save your changes and quit.

`gpg> save`


Customizing your Key
====================

We have a master key on a smartcard. Fantastic. Now it's time to to generate some subkeys with the `Sign`, `Encrypt` (really decrypt), and `Authenticate` capabilities and put them on a *different* smartcard. Then we'll polish things up and make any other tweaks this key needs.

`amnesia@amnesia:~$ gpg --expert --edit-key CDCD72AF`

~~~~~
Secret key is available.

pub  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06  usage: SC   
                     trust: unknown       validity: unknown
[ unknown] (1). Ada Lovelace <ada@enchantressofnumbers.net>

gpg> 
~~~~~

Additional UIDs
---------------

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

Photo UIDs
----------

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
pub  3072R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06  usage: SC   
                     trust: unknown       validity: unknown
[ unknown] (1)  Ada Lovelace <ada@enchantressofnumbers.net>
[ unknown] (2). Ada Lovelace <ada@analyticalengine.com>
[ unknown] (3)  [jpeg image of size 27348]
~~~~~


Subkey Generation
=================

Now it's time to make some subkeys. These are the keys you'll be using every day. We're going use a smartcard to store your everyday keys so that you don't actually need to tell your everyday computer what their secret parts are.


Subkey Type, Capabilities, & Size
---------------------------------

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

Since smartcards only go up to 3072 bits, that's our maximum.


Subkey Expiry
-------------

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

Since you're using a smartcard, you might not want your subkeys to expire: you can just keep using them until something goes wrong, and generate new ones then. If you're not using a smartcard, you might want to pick a reasonable expiry period. You have use your master key  to change subkeys, so you might want to make it a large fraction of your master key expiry period, and take that opportunity to rotate subkeys and postpone your master key's expiration at the same time.

~~~~~
Key is valid for? (0) 
Key does not expire at all
Is this correct? (y/N) y
Really create? (y/N) y
~~~~~

While you're waiting for GPG to gather entropy for this subkey, perhaps you might want to browse <http://entropykey.co.uk>. A UK company called Simtek Electronics makes a USB device which they claim generates truly random quantum entropy. Both the firmware and hardware are closed, so it's hard to verify that claim. Still, the open source MIT-licensed software is available from the site, and included in Debian and Ubuntu.

Repeat
------

Now we've got a subkey that can `Sign`. Repeat these steps two more times to generate an `Encrypt` and an `Authenticate` subkey, and you should get something that looks like this.

~~~~~
pub  4096R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06  usage: SC   
                     trust: unknown       validity: unknown
sub  3072R/FACBBB45  created: 2063-04-06  expires: never       usage: S   
sub  3072R/8FE97F11  created: 2063-04-06  expires: never       usage: E   
sub  3072R/94C86525  created: 2063-04-06  expires: never       usage: A   
[ unknown] (1). Ada Lovelace <ada@enchantressofnumbers.net>

gpg>
~~~~~


Finish Up
---------

We have our keys all set. They're the right lengths, there are the right number of them, they do the right things, they have the right UIDs. Everything looks great.

~~~~~
pub  3072R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06  usage: SC   
                     trust: unknown       validity: unknown
sub  3072R/FACBBB45  created: 2063-04-06  expires: never       usage: S   
sub  3072R/8FE97F11  created: 2063-04-06  expires: never       usage: E   
sub  3072R/94C86525  created: 2063-04-06  expires: never       usage: A   
[ unknown] (1)  Ada Lovelace <ada@enchantressofnumbers.net>
[ unknown] (2). Ada Lovelace <ada@analyticalengine.com>
[ unknown] (3)  [jpeg image of size 27348]
~~~~~

If you're happy with the way things look, you can save your changes and quit:

`gpg> save`

If you quit without saving, all your changes in this session will be discarded.


Subkey Backup
-------------

Our subkeys are ready, so let's back them up. Just like the master key, if we ever break or loose the smartcard they're stored on, it'd be nice to be able to restore them so we can still decrypt older messages. We also might one day want to change subkeys, and this backup allows us to re-use their smartcard, rather than having to keep it forever because it's the only copy of the subkeys.

When we export our secret-keys now, GPG will export the sub of our master key (with a note that it's on a smartcard), and the full private parts of our subkeys.

~~~~
amnesia@amnesia:~$ gpg --armor --output /media/cold-storage/ada-master_stub+secret-subkeys.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/cold-storage/ada-master_public+subkeys.gpg --export CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-alpha/ada-master_stub+secret-subkeys.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-alpha/ada-master_public+subkeys.gpg --export CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-omega/ada-master_stub+secret-subkeys.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-omega/ada-master_public+subkeys.gpg --export CDCD72AF
~~~~

We just made a backup so make sure to test it. Delete your keyrings or reboot, then re-import before continuing. If you reboot, remember to edit your `gpg.conf` again.


Subkeys on a Smartcard
======================

We've backed up the subkeys, so now it's time to put them on their own smartcard. Smartcards have three slots: one for each capability, so they can all fit on the same card. make sure not to mix up the card for your master key and the one for your subkeys!

You just set up a smartcard, so do the same with this one: change the PINs and add whatever personal info you want. Once you're ready, we can copy the subkeys over.

~~~~~
amnesia@amnesia:~$ gpg --edit-key CDCD72AF
Secret key is available.

pub  3072R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06  usage: SC   
                     trust:  unknown      validity:  unknown
sub  3072R/FACBBB45  created: 2063-04-06  expires: never       usage: S   
sub  3072R/8FE97F11  created: 2063-04-06  expires: never       usage: E   
sub  3072R/94C86525  created: 2063-04-06  expires: never       usage: A   
[ unknown] (1)  Ada Lovelace <ada@enchantressofnumbers.net>
[ unknown] (2). Ada Lovelace <ada@analyticalengine.com>
[ unknown] (3)  [jpeg image of size 27348]
~~~~~

We need to export keys to a card one at a time. Let's start with the signing key. First we need to toggle to the secret key listing.

~~~~~
gpg> toggle

sec  3072R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06
                     card-no: 0005 00000101
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

sec  3072R/CDCD72AF  created: 2063-04-06  expires: 2063-10-06
                     card-no: 0005 00000101
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
                     card-no: 0005 00000101
ssb  3072R/FACBBB45  created: 2063-04-06  expires: never     
                     card-no: 0005 00000102
ssb  3072R/8FE97F11  created: 2063-04-06  expires: never     
                     card-no: 0005 00000102
ssb  3072R/94C86525  created: 2063-04-06  expires: never     
                     card-no: 0005 00000102
(1)  Ada Lovelace <ada@enchantressofnumbers.net>
(2)  Ada Lovelace <ada@analyticalengine.com>
(3)  [jpeg image of size 27348]
~~~~~

The smartcard is ready to go. Save your changes and quit.

`gpg> save`


Everyday Export
===============

At this point, we've generated four keys (one master key and three subkeys) and exported each of them to a smartcard. This means that the "private" keys in this keyring are actually *all* just pointers to smartcards. Now we can export the stubs for use on our everyday machine. Even though we're using the `export-secret-keys` command, this output will actually contain *no* secret keys at all!

Backup
------

As always, let's back up our work before proceeding:

~~~~
amnesia@amnesia:~$ gpg --armor --output /media/cold-storage/ada-master_stub-complete.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-alpha/ada-master_stub-complete.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/key-backup-omega/ada-master_stub-complete.gpg --export-secret-keys CDCD72AF
~~~~

After a backup, delete your keyrings or reboot, then make sure you can import from any of those files.

"Keys"
------

Now export the key files for your everyday machine.

~~~~
amnesia@amnesia:~$ gpg --armor --output /media/everyday-usb/ada-everyday_stub.gpg --export-secret-keys CDCD72AF
amnesia@amnesia:~$ gpg --armor --output /media/everyday-usb/ada-everyday_pub.gpg --export CDCD72AF
~~~~

Revocation Certificates
-----------------------

Now's a perfectly good time to generate a revocation certificate. You might want to put this on your everyday machine, or you might want to print it out. A revocation certificate allows you to cancel your key without needing to have the secret part on hand (you used the secret part to create the certificate).

A jerk who got a hold of a revocation certificate for your key could also cancel your key (which would probably make a lot of work for you), but they couldn't sign things on your behalf, or decrypt secret messages meant only for you.

This probably means that you should store your revocation certificate somewhere other than where you store your master key backus, and just a little more accessible, so that even if you loose those backups, you can still revoke the key.

Here's how to make one.

~~~~
amnesia@amnesia:~$ gpg --armor --output /media/print-share/ada-revoke_misc.gpg --gen-revoke CDCD72AF

sec  3072R/CDCD72AF 2063-04-06 Ada Lovelace <ada@enchantressofnumbers.net>

Create a revocation certificate for this key? (y/N) y
Please select the reason for the revocation:
  0 = No reason specified
  1 = Key has been compromised
  2 = Key is superseded
  3 = Key is no longer used
  Q = Cancel
(Probably you want to select 1 here)
Your decision? 0
Enter an optional description; end it with an empty line:
>
Reason for revocation: No reason specified
(No description given)
Is this okay? (y/N) y
~~~~

Then you'll have to enter your user PIN. Make sure you're using the right smartcard, and you've got the right user PIN for it.

If you want, you can generate certificates for other scenarios too, but there's probably no need.


Finishing
=========

We're done with key generation now. If you've been following through all the steps, you probably have two smartcards/readers and five USB thumb drives:

* cold storage
* key backup alpha
* key backup omega
* everyday usb
* print share

The cold storage and key-bakup disks should have a full copy of the public and private parts of your master key and all your subkeys. Put these in your home safe or your piggy bank, or wherever you keep things safely. They should contain five files:

* `keyid_private.gpg`: the private part of your master key
* `keyid_public.gpg`: the public part of your master key
* `keyid_public+subkeys.gpg`: the public part of your master key and your subkeys
* `keyid_stub-complete.gpg`: the "private" pointers to smartcards for your master key and your subkeys
* `keyid_stub+private-subkeys`: the "private" pointer to your master key, and the actual private parts of your subkeys

The thumb drive destined for your everyday computer should have two files:

* `keyid_pub.gpg`: the public parts of all your keys
* `keyid_stub.gpg`: the "private" pointers to smartcards for all your keys

The print/share thumb-drive should have as many revocation certificates as you decided to make. Perhaps you should print them, or leave them with a trustworthy friend?

If you've got all that, you're ready to go. You can shut down your *secure computer* now. Next, we'll walk through setting up your everyday computer.


Everyday Setup & Usage
=======================

`gpg.conf` {#config}
----------

First, let's get GPG all snazzy-like. Here's a sample `gpg.conf` for you.

~~~~
#default-key <your long keyid>
#trusted-key <your long keyid>
#hidden-encrypt-to  <your long keyid>
default-recipient-self

ask-cert-level
auto-check-trustdb
no-greeting
no-expert

#cert-policy-url http://yoursite.net/id.txt (you can make this later)

auto-key-locate keyserver cert pka
keyserver hkp://pool.sks-keyservers.net

list-options no-show-photos show-uid-validity no-show-unusable-uids no-show-unusable-subkeys show-keyring show-policy-urls show-notations show-keyserver-urls show-sig-expire 
verify-options show-uid-validity
fixed-list-mode
keyid-format 0xlong

personal-digest-preferences SHA512
personal-cipher-preferences AES256 AES192 AES
cert-digest-algo SHA512
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed

s2k-cipher-algo AES256
s2k-digest-algo SHA512
s2k-mode 3
s2k-count 65011712

completes-needed 2
marginals-needed 5
max-cert-depth 7
min-cert-level 2
~~~~


Key Import
----------

Now Let's import those keys.

~~~~
ada@thinkpad:~$ gpg --import /media/everyday-usb/ada-everyday_pub.gpg
gpg: key 0x4771324ACDCD72AF: public key "Ada Lovelace <ada@enchantressofnumbers.net>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
~~~~

~~~~
ada@thinkpad:~$ gpg --import /media/everyday-usb/ada-everyday_stub.gpg
gpg: key 0x4771324ACDCD72AF: secret key imported
gpg: key 0x4771324ACDCD72AF: public key "Ada Lovelace <ada@enchantressofnumbers.net>" imported
gpg: Total number processed: 1
gpg:              unchanged: 1
gpg:       secret keys read: 1
gpg:   secret keys imported: 1
~~~~

Don't worry if you see a warning about time warps or clock problems. It probably means you were using a *secure computer* with a live OS like TAILS, and it didn't have an accurate clock.

It's also notable that things look a little different here. Instead of that short 8-character keyID, we have a longer 16-character one. That's because the display options in `gpg.conf`. It's always better to use longer keyIDs, they're harder to spoof. You should copy and paste this long keyID into your `gpg.conf` where there's a space for it.

If you want to, you gan `gpg --edit-key 4771324ACDCD72AF` and check that you can `toggle` to see that your everyday machine knows about the smartcards.

Uploading to a keyserver
------------------------

If everything still looks good, now you can upload your key to a keyserver. Keyservers never forget. Only do this once you're sure.

~~~~
ada@thinkpad:~$ gpg --send-keys 4771324ACDCD72AF
gpg: sending key 0x4771324ACDCD72AF to hkp server pool.sks-keyservers.net
~~~~

In a few minutes, your key should propagate to every keyserver out there.

Printing Fingerprints
---------------------

If you want other folks to sign your key, you'll need to print some pieces of paper with your key's fingerprint. The easiest way is to take this output and just copy and paste it into a text file a bunch of times.

~~~~
ada@thinkpad:~$ gpg --fingerprint 4771324ACDCD72AF
pub   3072R/0x4771324ACDCD72AF 2063-04-06 [expires: 2063-10-06]
      Key fingerprint = DDC6 93BF 8FC1 3036 36D2  CCFB 4771 324A CDCD 72AF
uid                 [ultimate] Ada Lovelace <ada@enchantressofnumbers.net>
uid                 [ultimate] Ada Lovelace <ada@analyticalengine.com>
uid                 [ultimate] [jpeg image of size 27348]
sub   3072R/0x8E550EC9FACBBB45 2014-02-13
sub   3072R/0xCA1659C68FE97F11 2014-02-13
sub   3072R/0xE9D0F1F494C86525 2014-02-13
