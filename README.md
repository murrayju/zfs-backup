This is forked from [adaugherity/zfs-backup](https://github.com/adaugherity/zfs-backup), and has been modified to work with [ubuntu-zfs](https://launchpad.net/~zfs-native/+archive/ubuntu/stable) ([ZFS on Linux](http://zfsonlinux.org/)) on Ubuntu (~14.10).

## About

This is a backup script to replicate a ZFS filesystem and its children to
another server via zfs snapshots and zfs send/receive over ssh.  It was
developed on Solaris 10 but should run with minor modification on other
platforms with ZFS support.

It supplements [zfs-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot), but runs independently.  I prefer that snapshots continue to be taken even if the backup fails.  It does not necessarily require that package -- anything that regularly generates snapshots that follow a given pattern will suffice.

This aims to be much more robust than the backup functionality of [zfs-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot), namely:
* it uses `zfs send -I` to send all intermediate snapshots (including any daily/weekly/etc.), and should still work even if it isn't run every hour -- as long as the newest remote snapshot hasn't been rotated out locally yet
* `zfs recv -dF` on the destination host removes any snapshots not present locally so you don't have to worry about manually removing old snapshots there.

#### What it does
1. Find the newest local daily snapshot
2. Find the newest remote daily snapshot (via ssh)
3. Check that the remote snapshot also exists locally
4. Use `zfs send -I` to send incremental updates from `$newest_remote` to `$latest_local` to the backup server
  * if anything fails, set put the service in maintenance mode and exit

## Command-line use
```
./zfs-backup.sh [options] [file.cfg]
```
#### available options
```
-n		debug/dry-run mode
-v		verbose mode
-f file	specify a configuration file
-r N		use the Nth most recent local snapshot rather than the newest
-h, -?	display help message
```

## Basic Installation
It is easiest to just run this script as a cron job. Before scheduling the cron job, be sure to follow the prerequisite config (below), and manually verify that your config works as expected.

> At the time of writing, ZoL only supports running `zfs` commands as the root user. Therefore, we must add this script to the root user's crontab, and connect to the remote server as root.

Create a new (daily executed) cron script
```
$ sudo su
# cd /etc/cron.daily
# vim zfs-backup-tank
```

Put the following script in that file
```
#!/bin/sh
exec /path/to/zfs-backup.sh -f /path/to/config/file.cfg
```

Make the script executable
```
# chmod +x zfs-backup-tank
```

zfs-backup supports commandline options and configuration files, so you can schedule different cron jobs with different config files and/or frequency, e.g. to back up to two different targets. If you schedule multiple cron
jobs, you should use different lockfiles in each configuration.

## Prerequisite Configuration

> At the time of writing, ZoL only supports running `zfs` commands as the root user. These instructions are based on that fact, and have to do some less-than-ideal things from a security standpoint. In the future (when ZoL supports it), these instructions should change to create a non-privileged user account instead of root.

1. [zfs-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot) or equivalent package installed locally, and regular snapshots generated (hourly, daily, etc.)
2. Set up ssh keys between root@localhost and root@remhost:
  * On the local machine

    ```
    $ sudo su
    # ssh-keygen
    # cat /root/.ssh/id_rsa.pub
    ```
  * Copy this output (the ssh public key), and add it to the authorized keys on the remote machine.
  
    ```
    $ sudo su
    # vim /root/.ssh/authorized_keys
    ```
  * On the remote machine, enable root login
    
    ```
    # vim /etc/ssh/sshd_config
    ```
    
    Append to the end of that file:
    ```
    PermitRootLogin yes
    RSAAuthentication yes
    PubkeyAuthentication yes
    ```
    
    Restart sshd to update with changes
    ```
    # service ssh restart
    ```
  * Test that key-based ssh works:
  
    ```
    # ssh root@remhost
    ```
3. Do an initial (full) zfs send/receive done so that remhost has the fs we are backing up, and the associated snapshots -- something like:

  ```
  zfs send -R $POOL/$FS@zfs-auto-snap_daily-(latest) | ssh root@$REMHOST zfs recv -dvF $REMPOOL
  ```
  Note: `zfs send -R` will send *all* snapshots associated with a dataset, so if you wish to purge old snapshots, do that first.
4. Create a config file to set the `TAG`/`PROP`/`REMHOST`/`REMPOOL` variables
  * See the `example.cfg` file
5. Set the property for each FS or volume that you want to include in the backup.

    > By default, $PROP is set to 'murrayju:backuptarget'. You can override this in the cfg file.
    
    ```
    zfs set $PROP=fullpath pool/fs
    ```
    `fullpath` will use `zfs recv -d`, meaning that `pool/a/b` will be replicated to `rempool/a/b`
    
    ```
    zfs set $PROP=basename pool/fs
    ```
    `basename` will use `zfs recv -e`, meaning that `pool/a/b` will be replicated to `rempool/b`. This is useful for replicating a sub-level FS into the top level of the backup pool.

## Troubleshooting

If a backup is not run for a long enough period, it is possible that the newest remote snapshot has been removed locally. In that case, you will have to manually run an incremental zfs send/recv to bring it up to date.
```
  zfs send -v -I zfs-auto-snap_daily-(latest on remote) -R $POOL/$FS@zfs-auto-snap_daily-(latest local) | ssh root@REMHOST zfs recv -dvF $REMPOOL
```
It's probably best to do a dry-run first `zfs recv -ndvF`.

Note: The default is to use daily snapshots because it is less likely that the snapshot you are using will be rotated out in the middle of a send. Also, note that ZFS will send all snapshots for a given filesystem before sending any for its children, rather than going in global date order.

Alternatively, use a different tag (e.g. weekly) that still has common snapshots, possibly in combination with the `-r` option (Nth most recent) to avoid short-lived snapshots (e.g. hourly) being rotated out in the middle of your sync. This is a good use case for an alternate configuration file.
