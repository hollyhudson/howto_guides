# Using a Yubikey for ssh

Here are the assumptions I am making about my audience:

- You use github or a similar online versioning and repository system.
- You have logged onto a remote server before so you are at least somewhat familiar with the process.
- You have at least a little experience on the command line.
- You're working on a Mac, or you know how to translate these steps for whatever operating system you're on.

If none of these are true, some of the discussion about how PGP and ssh work will still be useful, but you will probably find it difficult to follow the actual procedures.

## Overview and preparation

First off, some big-picture concepts.  You may have ssh'd into a machine using a public/private key pair generated with `ssh-keygen` instead of a username and password.  You may also have sent encrypted emails with a PGP public/private key pair generated with (usually) GPG Suite.  Both PGP encryption and ssh use RSA to verify identities, so they're kind of interchangeable despite the fact that we usually generate the keys with different tools for the two different purposes.

And that's a good thing because while `ssh-keygen` won't work with a Yubikey, GPG Suite does. So this guide is going to show you how to generate a key pair that you can use for ssh using GPG Suite, and how to put all the pieces in the right places so it will actually work.  It was born out of an afternoon of cursing at GPG Suite, which assumes you already know what you're doing and just throws a bunch of questions at you with no explanation of what reasonable settings might be.  I did a lot of research and asked friends a lot of questions, and hopefully this document will save you, and future me, a lot of confusion and headaches.

This will only work with a Yubikey that supports "OpenPGP".  You can check [Yubico's product charts](https://www.yubico.com/products/yubikey-hardware/compare-yubikeys/) to see which ones those are.

You'll need GPG Suite, so if it's not installed download it from [https://gpgtools.org](https://gpgtools.org).

## Generating a key pair

GPG tools have an idea of a "smart card" that can store your private and public keys.  The yubikey has the functionality to act like a smart card, so you'll be using GPG Suite's card editing tools to generate, query, import, and export keys.  So, put your Yubikey in your machine, open a terminal, and run:

```
gpg --card-edit
```

_(Note: If you already have gpg-agent running you'll get an "operation not supported" error.  Just kill gpg-agent and try the card edit command again.)_

This will drop you into an interactive command line program.  If you want to see all the possible commands enter `help`.

The first thing you'll probably want to do is change the PINs.  The defaults they start with are:

```
user: 123456
admin: 12345678
```

The PINs must be a minimum of 6 and and 8 digits, respectively.  As far as I can tell, you cannot change this and you cannot disable the requirements for a PIN.  The PIN is there to protect your card so that if someone gets ahold of your physical card (ie. Yubikey) they can't unlock it and use your keys or edit the card without knowing the PIN.  You will have to type the PIN whenever you use your Yubikey to encrypt or sign a file or email, and also when you use it to ssh or do your github pushes.  Make the PINs as difficult or easy to type as your particular threat model requires.  To change the admin one you'll need to enter `admin` at the `gpg/card>` prompt, and then:

```
gpg/card> passwd
```

Follow the prompts and menus to make the changes.  Don't rush.. read the dialogs.  Sometimes they want the admin PIN, sometimes they want the user PIN.

Now we're ready to generate a key.  If you want to make a key larger than 2048 bits (so if you want to make one that is 4096 bits) run `key-attr` to edit the key attributes.  Select RSA for each (ECC won't work on the Yubikey) and type the key size you want (4096 is a reasonable choice).

Now you can actually generate the key by running `generate`.  You're going to get a bunch of questions that are going to seem weird if you haven't done pgp stuff recently.  GPG Suite assumes you're generating these keys to encrypt your files and communications, so these questions are to fill out the data that gets sent to the key server if you upload your public key.  If you're just using this key pair for ssh and github, none of the answers you put matter.  If you're think -- well hey, maybe I will also use this for email, then here's what you need to know.

First off, the info you enter will be associated with your public key as metadata.  It won't affect the keys themselves in any way, and will only go public if you upload your public key to a keyserver.  And now for the questions:

### Off-card back up

_NOTE: This is your only chance to get an off-card copy of your private key.  If you want to do this, you have to say yes NOW.  You won't get another chance._

This one is really going to be personal preference.  If you decide to create an off-card back up you'll be prompted for a passphrase, and that passphrase will be required to actually use that exported key (but not the one still on the card), so it's not like the exported private key will just be sitting there completely exposed to anyone who gains entry to your system -- they would still have to know your passphrase.  But of course your passphrase would need to be long and complex enough to not be crackable.  And you would want to be sure there are no keyloggers installed on your machine that could capture your passphrase.  Right.. this is probably why we wanted to put this thing on hardware in the first place -- because it keeps off a machine that is always connected to the internet onto a device that you, presumably, have more control over.  But, you know, do what's right for your threat model.

### Expiration

The expiration date you set won't affect the keys themselves at all, since the Yubikey has no sense of time, no ability to self-destruct or self-delete contents, and it doesn't talk to the keyserver.  The expiration date only comes into play if you upload your key to the keyserver, and then it's only used as a message to the public that they should not encrypt communications for you with this key anymore, and should not trust new documents signed with this key.  When you generate a key you will also get a revocation file, which you can use to revoke the key -- but it works the same way as key expiration.  You would send the revocation file to the key server, and it would mark the key as no longer in use by you, but the key itself isn't going to stop working.  (So if you do lose your Yubikey, you definitely want to remove its public key from github and delete its entries from the `.ssh/authorized_keys` files on your servers to keep anyone who finds it from logging into your stuff.)

So, tl;dr: 

- If you plan to never use this key pair for encrypting communications, you can set it to expire tomorrow, it really doesn't matter.  Or just put whatever and and never push your public key to the keyserver.
- Or, if you do plan to use it for encrypting communications, you can set it for far in the future and if you lose your yubikey next week you can just revoke it.
- Or, if you again want to use it for communications and you want to set a "just in case" expiration after a few years to force yourself to make a new key, that's also fine.

### Name, email, and comment

Again, this is only going to the keyserver, and only if you upload your public key.  If you don't plan to use this key pair for encrypting communications or files, and only for ssh and github, just put whatever and move on.  

If you do want to use this key pair for encrypting communications, then you have some choices:

- If you want others to easily find you and your key (for instance if you are a journalist), make this information as accurate as possible, and maybe add a comment that provides more contact info or ways of verifying your key.  And while you're at it, if you have an email that you use with a mail client that makes decrypting PGP easy, you probably want to use that email address.

- If you don't like the fact that having a key on a keyserver and the key signing that it facilitates helps others discern your social graph, or if you're only using it to communicate secretly with people you can exchange keys directly with, then you're probably not using the keyserver anyway so feel free to put whatever.  

## Confirmation and writing

The last question just asks you to confirm your choices, and then the card starts generating and writing the key.  This takes a while, but you'll know the Yubikey is still working because it will be flashing its LED.   Do some knitting or read some XKCD while you wait.

Ok, now you have keys!  ..probably.. let's make sure:

```
gpg/card> list
```

If it returns info on the key, we can move on.  Well, actually, first take note of the key ID, which is listed on this line:

```
General key info..: pub  rsa4096/[key ID here]
```

You'll want that for the next step.

## Getting SSH to work with the new key

Now, ssh-agent is not equiped to the handle the pgp key we just created, so we we want to have gpg-agent take over for ssh-agent whenever we need to use the key.  In `.gnupg/gpg-agent.conf` add the following line:

```
enable-ssh-support
```

And we need configure our shell (bash in this case) to tell ssh where to look for the gpg agent when it needs to deal with keys.  We also need to make sure gpg-agent starts up if it's not already running when we open a shell.  Here are the lines to add to your `.bash_profile`:

```
export GPG_TTY=$(tty)
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

Source your `.bash_profile` or log into a new terminal.

Now you should be able to list your keys with:

```
ssh-add -l
```

If you see the key you just created (`cardno:` should be followed by the number printed on the back of your Yubikey), then everything worked.  We can now export the public key and copy it wherever we need it.  

Ok, remember when I told to you take note of your key ID?  We're going to use it now (don't type the brackets):

```
gpg --export-ssh-key [key ID] 
```

Copy-paste the public key that is returned into an appropriately-named file, something like `rsa_id.yubikey123.pub` in your `.ssh` directory, where 123 is replaced by the number on the Yubikey (useful if you have more than one Yubikey, and you probably should have at least one for backup).  You can copy that `.pub` file to the `.ssh/authorized_keys` files on your servers and to github.

**Test and make sure it works!**

If you want to require touch for decryption, there's one more step.

Install Yubico's `ykman` tool (with brew or whatever you use for package management).  Then:

```
ykman openpgp touch aut on
ykman openpgp touch sig on
ykman openpgp touch enc on
```

That's it!  Well almost.  There are some files in your `.gnupg` directory that are necessary to tell the gpg tools where to find your key.  (Confession time: I don't actually know _which_ files those are, I just know they're in there.)  This means that if your computer dies or you sometimes work from a second machine, the yubikey is not going to "just work" unless you've have a copy of your `.gnupg` directory on the other machine.  And you should probably keep a back-up of it somewhere off of your main machine in case that computer dies.

If you have suggestions for something I should add or change, make an issue or send a pull request :)
