---
date:  2024-06-12
tags:
  - email
  - neomutt
---

# Archive old Email

## Preconditions

My root mail folder is `~/.local/share/mail`, this is also where
`notmuch config get database.path` points to. My private email is stored in a
folder "tobi.pleyer@gmail.com". In there I created a folder called "Archive"
and in that folder I created a directory for every year I need:

```bash
$ mkdir ~/.local/share/mail/tobi.pleyer@gmail.com/Archive
$ cd ~/.local/share/mail/tobi.pleyer@gmail.com/Archive
$ mkdir 20{12,13,14,15,16,17,18,19,20,21,22,23}
$ for d in 20*; do pushd $d; mkdir cur; mkdir new; mkdir tmp; popd; done
```

## Moving mails to archive and delete them from GMail

I have the following workflow within neomutt to archive the mails by year and
mark them as deleted to GMail. I use notmuch for the queries:

```
\\ (custom notmuch query)
query: date:2012-01..2012-12
T. (tag all)
Cc (bound to macro index,pager Cc ";<copy-message>" "copy mail")
=Archive/2012<enter>
;D (delete all tagged)
```

This copies the messages to the archive and deletes them. The deletion moves
them to the trash, so that they are properly cleaned by GMail.

## Move archive to backup

```bash
$ rsync -avh --progress /home/tobias/.local/share/mail/tobi.pleyer@gmail.com/Archive/ /mnt/smb/nas/tobias/Email/Archive/
```
