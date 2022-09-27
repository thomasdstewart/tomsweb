---
title: "KeepassXC as Secret Service"
summary: ""
authors: ["thomas"]
tags: ["linux", "keepass", "debian"]
categories: []
date: 2022-07-22 09:37:00
---
When running the Gnome Desktop Environment on Debian there is a secrets tool that it automatically run automatically called Gnome Keyring. This tools provides multiple functions:

 * ssh keys - ssh keys in ~/.ssh with passwords that match the login password are unlocked at login time via pam and added to a ssh-agent (the gnome keyring agent not the original openssh agent one)
 * general secrets via dbus - the secret service is accessed via dbus. The secrets are stored in an encrypted file (~/.local/share/keyrings), this file is also unlocked at login time via pam. The secrets are available via libsecret. So for example nextcloud login passwords are stored in this store. Also if it's not been unlocked at login time, for instance if login via fingerprint is used access via dbus starts the gnome-keyring-daemon and the user is prompted for the password.
 * pki - gnome-keyring also stores pki certificates, I've not used this much, but I assume its a location to store certificates: custom CA's, machine and user certificates.

There are many tools to them use the above, seahorse is a gui tool and secret-tool is a cli tool.

The idea being that ssh key passwords and general passwords are stored in encrypted files on disk, albeit with the same password as login.

## The Problem

* ssh keys only when keepass unlocked
* ansbile secerts, aws secrets,
* easier to keep an handle on

## URLs
https://wiki.archlinux.org/title/GNOME/Keyring
https://gitlab.freedesktop.org/xdg/xdg-specs/-/issues/75
https://github.com/keepassxreboot/keepassxc/issues/6274
https://rtfm.co.ua/en/what-is-linux-keyring-gnome-keyring-secret-service-and-d-bus/
