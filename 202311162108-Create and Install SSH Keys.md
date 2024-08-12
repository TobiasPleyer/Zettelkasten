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

A good explanation of SSH and its authentication variants can be found in this
[Illustrated Guide to SSH Agent Forwarding](http://www.unixwiz.net/techtips/ssh-agent-forwarding.html)
