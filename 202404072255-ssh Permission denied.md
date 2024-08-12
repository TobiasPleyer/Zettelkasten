---
date:  2024-04-07
tags:
  - proxmox
---

# Ssh Permission denied

There are many possible reasons for this error, but my problem was that I
switched a previously unpriviledged container to priviledged.

The consequence was that many (important!) files had still the ownership
`100000:100000`, previously mapped to root but now an unknown user.

This made `sudo` unavailable and somehow messed up PAM for the ssh login.

Steps to fix:

```bash
$ chown 0:0 /etc/sudo.conf
$ chmod u+s /usr/bin/sudo # sets the setuid bit
$ chown --from=100000:100000 0:0 -R /
```

The problem showed itself when trying to use sudo:

```bash
$ sudo mount -o remount,rw /
sudo: /etc/sudo.conf is owned by uid 100000, should be 0
sudo: /usr/bin/sudo must be owned by uid 0 and have the setuid bit set
```

Optionally renew the password for root

```bash
$ sudo passwd root
```
