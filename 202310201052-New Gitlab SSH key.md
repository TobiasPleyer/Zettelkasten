---
date:  2023-10-20
tags:
  - howto
---

# New Gitlab SSH key

When the expiration warning comes:

```
❯ git pull
remote:
remote: INFO: Your SSH key is expiring soon. Please generate a new key.
remote:
...
```

It is time to renew the SSH key

```
❯ ssh-add -L
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGDDOeUSoEmTl75K3udQXww48xyDyJsMnTNDgT58kcmB tpleyer@heilmannsoftware.de
...
# Delete the key from the ssh agent
❯ ssh-add -d ~/.ssh/heilmannsoftware
# Delete the key
❯ rm heilmannsoftware*
# Create a new key
❯ ssh-keygen -t ed25519 -C "tpleyer@heilmannsoftware.de"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/tobias/.ssh/id_ed25519): /home/tobias/.ssh/heilmannsoftware
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/tobias/.ssh/heilmannsoftware
Your public key has been saved in /home/tobias/.ssh/heilmannsoftware.pub
The key fingerprint is:
SHA256:bL2QAalYsQ1BWmdxKzBK/jqbBjNnKlYKqUFvDN+H1/k tpleyer@heilmannsoftware.de
The key's randomart image is:
+--[ED25519 256]--+
|  ..Oo=o.        |
| o +.O.o .       |
|  +o..o o        |
| o...  o +       |
|..= o . S o      |
|B oB o + + .     |
|oO*   o   o      |
|+o.+       E     |
|o.o              |
+----[SHA256]-----+
# Add the key to the ssh agent
❯ ssh-add ~/.ssh/heilmannsoftware
Identity added: /home/tobias/.ssh/heilmannsoftware (tpleyer@heilmannsoftware.de)
```
