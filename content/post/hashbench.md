---
title: "Hash benching"
summary: ""
authors: ["thomas"]
tags: ["linux", "crc", "md5", "sha"]
categories: []
date: 2025-04-04 10:38:00
---





```
#!/usr/bin/env python3
import zlib

file_path='/home/thomas/test'
crc32_hash = 0
with open(file_path, 'rb') as f:
    while chunk := f.read(8192):
        crc32_hash = zlib.crc32(chunk, crc32_hash)
print(format(crc32_hash & 0xFFFFFFFF, '08x'))
```


dd if=/dev/zero bs=1M of=/home/thomas/test count=40960


$ time ./crc.py 
e38a6876

real	0m46.361s
user	0m21.866s
sys	0m18.824s


$ time crc32 test
e38a6876

real	0m44.175s
user	0m19.582s
sys	0m19.385s




```
#!/usr/bin/env python3
import hashlib

file_path='/home/thomas/test'
with open(file_path, 'rb') as f:
    md5 = hashlib.md5()
    while chunk := f.read(8192):
        md5.update(chunk)
print(md5.hexdigest())
```


$ time ./md5.py 
c45e93a611a7283b3be8a261b4c801b6

real	1m30.876s
user	1m12.638s
sys	0m17.234s



$ time md5sum test
c45e93a611a7283b3be8a261b4c801b6  test

real	1m25.931s
user	1m7.712s
sys	0m18.013s




thomas@ekanite(degraded) ~ (130)8768$ time sha1sum test 
37ca1826b64b9fa14a9893f040c593c69a9a90ad  test

real	0m47.863s
user	0m30.807s
sys	0m16.670s
thomas@ekanite(degraded) ~ 8769$ time sha224sum test 
6bfbaf887b888fe307d551cba8b2b8de16ca8f80a59f6069d89b6a0b  test

real	0m52.538s
user	0m35.996s
sys	0m16.324s
thomas@ekanite(degraded) ~ 8770$ time sha512sum test 
68eaa567f0ede602c8a89bae07093d42afa5bb42306c99c2a9f2c124d688e42e323bae405b3ca06f5dc360d13325159e09e2ab89a9c82822356e25344fadc787  test

real	1m15.987s
user	1m0.268s
sys	0m15.133s
thomas@ekanite(degraded) ~ 8771$ time sha256sum test 
2109856cb6642099b7ae3ee3bdf2b1bd7f64af573b04958e8cdd278a786252cc  test

real	0m40.010s
user	0m27.751s
sys	0m12.037s
thomas@ekanite(degraded) ~ 8772$ 








When running the Gnome Desktop Environment on Debian there is a secrets tool that automatically runs called Gnome Keyring. This tools provides multiple functions:

 * ssh keys - ssh keys in ~/.ssh with passwords that match the login password are unlocked at login time via pam and added to a ssh-agent (the gnome keyring agent not the original openssh agent one)
 * general secrets via dbus - the secret service is accessed via dbus. The secrets are stored in an encrypted file (~/.local/share/keyrings), this file is also unlocked at login time via pam. The secrets are available via libsecret. So for example nextcloud login passwords are stored in this store. Also if it's not been unlocked at login time, for instance if login via fingerprint is used access via dbus starts the gnome-keyring-daemon and the user is prompted for the password.
 * pki - gnome-keyring also stores pki certificates, I've not used this much, but I assume its a location to store certificates: custom CA's, machine and user certificates.

There are many tools to them use the above, seahorse is a gui tool and secret-tool is a cli tool.

The idea being that ssh key passwords and general passwords are stored in encrypted files on disk, albeit with the same password as login.

## The Problem

* hanging passwords is hard, eg have to update password for ~/.local/share/keyrings
* logging in via fingerprint
* ssh keys only when keepass unlocked
* ansbile secerts, aws secrets, etc end up in ~/.local/share/keyrings across laptops
* easier to keep an handle on in just keepass files

## Mods
Notes on changes
```
systemctl --user mask gnome-keyring-daemon.service
systemctl --user mask gnome-keyring-daemon.socket

.config/autostart/gnome-keyring-pkcs11.desktop
.config/autostart/gnome-keyring-secrets.desktop
.config/autostart/gnome-keyring-ssh.desktop

pam-auth-update remove gnome-keyring

/etc/pam.d/gdm-password


/home/thomas/.local/share/dbus-1/services/org.freedesktop.impl.portal.Secret.service:[D-BUS Service]
/home/thomas/.local/share/dbus-1/services/org.freedesktop.impl.portal.Secret.service:Name=org.freedesktop.impl.portal.Secret
/home/thomas/.local/share/dbus-1/services/org.freedesktop.impl.portal.Secret.service:Exec=/usr/bin/keepassxc
/home/thomas/.local/share/dbus-1/services/org.freedesktop.secrets.service:[D-BUS Service]
/home/thomas/.local/share/dbus-1/services/org.freedesktop.secrets.service:Name=org.freedesktop.secrets
/home/thomas/.local/share/dbus-1/services/org.freedesktop.secrets.service:Exec=/usr/bin/keepassxc
/home/thomas/.local/share/dbus-1/services/org.gnome.keyring.service:[D-BUS Service]
/home/thomas/.local/share/dbus-1/services/org.gnome.keyring.service:Name=org.gnome.keyring
/home/thomas/.local/share/dbus-1/services/org.gnome.keyring.service:Exec=/usr/bin/keepassxc

```

## URLs
https://wiki.archlinux.org/title/GNOME/Keyring
https://gitlab.freedesktop.org/xdg/xdg-specs/-/issues/75
https://github.com/keepassxreboot/keepassxc/issues/6274
https://rtfm.co.ua/en/what-is-linux-keyring-gnome-keyring-secret-service-and-d-bus/
