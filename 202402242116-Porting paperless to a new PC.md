---
date:  2024-02-24
tags:
  - howto
---

# Porting paperless to a new PC

Until now I always had paperless ngx running on my synology. However I am not
satisfied with the performance. The whole interface feels slow, but the absolute
deal breaker is the fact that it takes forever to load documents in the bulk
edit mode.

So I decided to move to a LXC container hosted on my proxmox server.

But how to do that?

## Paperless

From the paperless side this is a fairly easy task:

- excute [document_exporter](https://docs.paperless-ngx.com/administration/#exporter)
  on the machine/container currently hosting paperless
- somwhow bring the backup files to the new machine
- run [document_importer](https://docs.paperless-ngx.com/administration/#importer)

## From Synology to Proxmox

### Synology

After running the export (backup) all files were available under /volume1/docker/paperless/export.

Copy to /volume1/backups/paperless_ngx

Make it a NFS share and allow access from 172.16.1.21 (The container's IP running on Proxmox)

### Proxmox

Normally containers are not allowed to use NFS (unprivileged). For reference see
[unix stackexchange](https://unix.stackexchange.com/questions/715278/mount-nfs-operation-not-permitted-in-proxmox-container)
and the [proxmox forum](https://forum.proxmox.com/threads/unable-to-use-nfs-share-within-lxc-container.58045/#post-267777).

So one has 2 options:

    1. Create a privileged container
    2. Mount in the Proxmox host and create a bind mount

I tried to do the latter. First the NFS mount on the host:

```bash
root@pve:~# mkdir /var/synology/backups
root@pve:~# mount -t nfs 172.16.1.131:/volume1/backups /var/synology/backups
```

And not I tried to bind mount the NFS folder into the container:

```bash
root@pve:~# pct stop 105
root@pve:~# pct set 105 -mp0 /var/synology/backups/paperless_ngx,mp=/var/backups
root@pve:~# pct start 105
```

This didn't help because the owner of the files was 'nobody', as explained in this
[wiki article](https://pve.proxmox.com/wiki/Unprivileged_LXC_containers) about how the uid and gui
are mapped between the host and the container for unprivileged LXC containers. However I couldn't get it to work...

After a lot of trying I found [this blog post](https://theorangeone.net/posts/mount-nfs-inside-lxc/)
suggesting to simply make the container privileged, which apparently mostly means that the uid and gui are not mapped
between the host and the container.

In the end this also didn't do the trick for me. For some reason the container started, but none of the expected
services (paperless, redis, etc.) was started. Simply making the container unpriviledged solved the problem, but of
then the NFS mapping was broken...

Finally I ended up simply using `scp` to copy the backup to the container.
