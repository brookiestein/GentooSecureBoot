# GentooSecureBoot
**My step-by-step guide to self-sign Linux kernel image on Gentoo to be able to boot with Secure Boot enabled.**

To make this guide, and also to self-sign my Linux kernel image, I read [Sakaki's EFI Install Guide/Configuring Secure Boot](https://wiki.gentoo.org/wiki/User:Sakaki/Sakaki%27s_EFI_Install_Guide/Configuring_Secure_Boot) 
and the [Gentoo's Secure Boot Wiki Page](https://wiki.gentoo.org/wiki/Secure_Boot#Signing_Boot_Files).
I just modified them a little bit to make it easier for me to work with this in case I need to do it again in the future.

## Requisites:
1. You need to have installed: `app-crypt/efitools`, and `app-crypt/sbsigntools`.
2. All the stuff we're going to do here, needs super-user permission, so log in with the root user in your terminal emulator or use `sudo`, `doas`, or any equivalent.

## Clear Existing Keys
The first thing I do is to clear the current keys from the BIOS/UEFI. Don't do that if you want to keep Microsoft's, which normally come pre-installed in a lot of laptops.

If you want to do that, I'll have to refer you to [Sakaki's EFI Install Guide/Configuring Secure Boot](https://wiki.gentoo.org/wiki/User:Sakaki/Sakaki%27s_EFI_Install_Guide/Configuring_Secure_Boot)
because I don't use Microsoft's keys. I just left my system with my own rather than having both Microsoft's and mine.

**It's useful to have both if you have dual-boot, though.**

## Create The Working Directory
I store all my keys in `/etc/efikeys`.

Make sure only `root` has access to this directory:
```
mkdir -v /etc/efikeys
chmod -v 700 /etc/efikeys
```

## Back up Existing Keys (Optional)
If you don't want to erase all the existing keys and append the ones we're going to create here instead, you can back them up by:
```
mkdir backups
cd backups
efi-readvar -v PK -o PK.esl.bak
efi-readvar -v KEK -o KEK.esl.bak
efi-readvar -v db -o db.esl.bak
efi-readvar -v dbx -o dbx.esl.bak
cd ..
```

## Create Our Own Keys
```
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=Brayan's Platform Key/" -keyout PK.key -out PK.crt -days 3650 -nodes -sha256
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=Brayan's Key Exchange Key/" -keyout KEK.key -out KEK.crt -days 3650 -nodes -sha256
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=Brayan's Kernel-signing Key/" -keyout db.key -out db.crt -days 3650 -nodes -sha256
```
Replace the Common Name (CN) with your name or anything you want, that won't make any difference for your computer, that's just a text that will help us identify our keys.

These keys expire in 10 years. I don't think I'll be using the same computer for more than 10 years, so for me that's pretty nice.

## Make Those Keys Only-readable by the root User
This isn't strictly mandatory since the Working Directory is already only-readable by the root user.
```
chmod -v 400 *.key
```

## Create the signature list (a.k.a .auth files)
This step isn't necessary for all BIOSes, but mine accepts installing the keys in that format, so I need them.
```
uuidgen > uuid.txt
cert-to-efi-sig-list -g `cat uuid.txt` PK.crt PK.esl
sign-efi-sig-list -k PK.key -c PK.crt PK PK.esl PK.auth

cert-to-efi-sig-list -g `cat uuid.txt` KEK.crt KEK.esl
sign-efi-sig-list -a -k PK.key -c PK.crt KEK KEK.esl KEK.auth
# (Notice that the PK private key was used to sign, in this case, and that we used the -a option,
# to indicate that this is to be used to append data, rather than replace it.)
# The file we need out of this is KEK.auth. 

cert-to-efi-sig-list -g `cat uuid.txt` db.crt db.esl
sign-efi-sig-list -a -k KEK.key -c KEK.crt db db.esl db.auth

sign-efi-sig-list -k KEK.key -c KEK.crt dbx dbx.esl.bak dbx.auth
```

## Installing the New Keys into the Keystore
At this point we should have no keys stored in the Keystore since we cleared them. To verify that, run:
```
efi-readvar

# Output:
# Variable PK has no entries
# Variable KEK has no entries
# Variable db has no entries
# Variable dbx has no entries
```
Cite Sakaki: You may see a fifth variable, MokList, displayed when you do this. This is an EFI "Boot Services Only Variable" which some Linux distributions use to allow their bootloader shims to work under secure boot. We won't need to worry about it.
### Installing the Keys (Really :)
```
efi-updatevar -e -f dbx.esl.bak dbx
efi-updatevar -e -f db.esl db
efi-updatevar -e -f KEK.esl KEK
```

Cite Sakaki: The -e option specifies that an EFI signature list file is to be loaded (and the -f option precedes the filename itself). Because we are in setup mode, no private key is required for these operations (which it would be, if we were in user mode). 

### Install the Platform Key (PK)
```
efi-updatevar -f PK.auth PK
```
I have problems with installing the PK this way, so I install it manually in the BIOS/UEFI. Nothing difficult, just:
```
cp PK.auth /boot/efi
```

`/boot/efi` will be visible in the BIOS/UEFI.

## Verify that the keys are installed
```
efi-readvar

# Output:
# Variable PK, length <#>
PK: List 0, type X509
    Signature 0, size <#>, owner <your key>
# KEK, db, and dbx output
```

## Signing the Linux Kernel Image
```
# CWD: /etc/efikeys
sbsign --key db.key --cert db.crt --output /boot/vmlinuz-6.6.21-gentoo /boot/vmlinuz-6.6.21-gentoo
```
Optionally you can tell `sbsign` not to override the unsigned image with the signed one by giving it another name, for example: `--output /boot/vmlinuz-signed-6.6.21-gentoo` or something like that.

Personally I don't care overwriting it because I don't want to have both the unsigned and the signed kernel image.

That's it! We already have our Linux kernel image signed, let's prove that:
```
sbverify --cert db.crt /boot/vmlinuz-6.6.21-gentoo

# Output:
# Signature verification OK
```

## Enable SecureBoot
Now just reboot your computer, and enable Secure Boot.

## Suggestion
As a suggestion, you can set a password for Supervisor on your computer, that way you can make it a little more difficult for an intruder to clear your keys or to disable secure boot on your computer.

That doesn't hurt, so I did it.

## More Information
For more information, please read [Sakaki's EFI Install Guide/Configuring Secure Boot](https://wiki.gentoo.org/wiki/User:Sakaki/Sakaki%27s_EFI_Install_Guide/Configuring_Secure_Boot) 
and the [Gentoo's Secure Boot Wiki Page](https://wiki.gentoo.org/wiki/Secure_Boot#Signing_Boot_Files).

This guide is just to make my job easier.

Thanks for reading my Step-by-Step Guide on Making My Own Linux Kernel Signed and being able to boot Gentoo Linux with Secure Boot enabled!
