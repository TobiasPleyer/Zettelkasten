---
date:  2024-02-26
tags:
  - howto
---

# Create a Samba share

Quick start guide.
[Original](https://help.ubuntu.com/community/How%20to%20Create%20a%20Network%20Share%20Via%20Samba%20Via%20CLI%20(Command-line%20interface/Linux%20Terminal)%20-%20Uncomplicated,%20Simple%20and%20Brief%20Way!)

```bash
$ sudo apt-get update
$ sudo apt-get install samba
$ sudo smbpasswd -a <user_name> # the user should exist on the system
$ mkdir /home/<user_name>/<folder_name>
$ sudo cp /etc/samba/smb.conf ~
$ sudo nano /etc/samba/smb.conf

Once "smb.conf" has loaded, add this to the very end of the file:

[<folder_name>]
path = /home/<user_name>/<folder_name>
valid users = <user_name>
read only = no

$ sudo service smbd restart
```
