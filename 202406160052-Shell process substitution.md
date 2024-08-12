---
date:  2024-06-16
tags:
---

# Shell process substitution

See the [Bash manual](https://tldp.org/LDP/abs/html/process-sub.html) for more insight.

Shell substitution allows to use a process where normally a file is expected.

Example: tee

tee's first argument should be a file. But we can also use it as a "split" to feed the input to two processes.

```bash
$ tee >(notmuch insert --folder=Sent +sent) | sendmail $*
```

In this example the mail arriving on stdin is sent to notmuch AND sendmail.
