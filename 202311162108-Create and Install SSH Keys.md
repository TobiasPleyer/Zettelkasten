---
date:  2023-11-16
tags:
  - linux
  - ssh
  - howto
---

# Create and Install SSH Keys

Generate a SSH key pair:

```
$ ssh-keygen
```

Install (copy) it on the remote machine

```
$ ssh-copy-id username@remote.machine
```
