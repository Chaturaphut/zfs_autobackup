# ZFS autobackup

Introduction
============

ZFS autobackup is used to periodicly backup ZFS filesystems to other locations. This is done using the very effcient zfs send and receive commands.

It has the following features:
* Works across operating systems: Tested with Linux, FreeBSD/FreeNAS and SmartOS.
* Works in combination with existing replication systems. (Like Proxmox HA)
* Automatically selects filesystems to backup by looking at a simple ZFS property. (recursive)
* Creates consistent snapshots. (takes all snapshots at once, atomic.)
* Multiple backups modes:
  * "push" local data to a backup-server via SSH.
  * "pull" remote data from a server via SSH and backup it locally.
  * Backup local data on the same server.
* Can be scheduled via a simple cronjob or run directly from commandline.
* Supports resuming of interrupted transfers. (via the zfs extensible_dataset feature)
* Backups and snapshots can be named to prevent conflicts. (multiple backups from and to the same filesystems are no problem)
* Always creates a new snapshot before starting.
* Checks everything but tries continue on non-fatal errors when possible. (Reports error-count when done)
* Ability to 'finish' aborted backups to see what goes wrong.
* Easy to debug and has a test-mode. Actual unix commands are printed.
* Keeps latest X snapshots remote and locally. (default 30, configurable)
* Uses zfs-holds on important snapshots so they cant be accidentally destroyed.
* Tries to work around quota issues by temporary clearing those properties during backup.
* Easy installation:
  * Only one host needs the zfs_autobackup script. The other host just needs ssh and the zfs command.
  * Written in python and uses zfs-commands, no 3rd party dependency's or libraries.
  * No separate config files or properties. Just one command you can copy/paste in your backup script.

Usage
====
```
usage: zfs_autobackup [-h] [--ssh-source SSH_SOURCE] [--ssh-target SSH_TARGET]
                      [--keep-source KEEP_SOURCE] [--keep-target KEEP_TARGET]
                      [--no-snapshot] [--no-send] [--allow-empty]
                      [--ignore-replicated] [--no-holds] [--ignore-new]
                      [--resume] [--strip-path STRIP_PATH] [--buffer BUFFER]
                      [--clear-refreservation] [--clear-mountpoint]
                      [--filter-properties FILTER_PROPERTIES] [--rollback]
                      [--ignore-transfer-errors] [--test] [--verbose]
                      [--debug]
                      backup_name target_path

ZFS autobackup v2.4

positional arguments:
  backup_name           Name of the backup (you should set the zfs property
                        "autobackup:backup-name" to true on filesystems you
                        want to backup
  target_path           Target ZFS filesystem

optional arguments:
  -h, --help            show this help message and exit
  --ssh-source SSH_SOURCE
                        Source host to get backup from. (user@hostname)
                        Default local.
  --ssh-target SSH_TARGET
                        Target host to push backup to. (user@hostname) Default
                        local.
  --keep-source KEEP_SOURCE
                        Number of days to keep old snapshots on source.
                        Default 30.
  --keep-target KEEP_TARGET
                        Number of days to keep old snapshots on target.
                        Default 30.
  --no-snapshot         dont create new snapshot (usefull for finishing
                        uncompleted backups, or cleanups)
  --no-send             dont send snapshots (usefull to only do a cleanup)
  --allow-empty         if nothing has changed, still create empty snapshots.
  --ignore-replicated   Ignore datasets that seem to be replicated some other
                        way. (No changes since lastest snapshot. Usefull for
                        proxmox HA replication)
  --no-holds            Dont lock snapshots on the source. (Usefull to allow
                        proxmox HA replication to switches nodes)
  --ignore-new          Ignore filesystem if there are already newer snapshots
                        for it on the target (use with caution)
  --resume              support resuming of interrupted transfers by using the
                        zfs extensible_dataset feature (both zpools should
                        have it enabled) Disadvantage is that you need to use
                        zfs recv -A if another snapshot is created on the
                        target during a receive. Otherwise it will keep
                        failing.
  --strip-path STRIP_PATH
                        number of directory to strip from path (use 1 when
                        cloning zones between 2 SmartOS machines)
  --buffer BUFFER       Use mbuffer with specified size to speedup zfs
                        transfer. (e.g. --buffer 1G) Will also show nice
                        progress output.
  --clear-refreservation
                        Set refreservation property to none for new
                        filesystems. Usefull when backupping SmartOS volumes.
                        (recommended)
  --clear-mountpoint    Sets canmount=noauto property, to prevent the received
                        filesystem from mounting over existing filesystems.
                        (recommended)
  --filter-properties FILTER_PROPERTIES
                        Filter properties when receiving filesystems. Can be
                        specified multiple times. (Example: If you send data
                        from Linux to FreeNAS, you should filter xattr)
  --rollback            Rollback changes on the target before starting a
                        backup. (normally you can prevent changes by setting
                        the readonly property on the target_path to on)
  --ignore-transfer-errors
                        Ignore transfer errors (still checks if received
                        filesystem exists. usefull for acltype errors)
  --test                dont change anything, just show what would be done
                        (still does all read-only operations)
  --verbose             verbose output
  --debug               debug output (shows commands that are executed)

When a filesystem fails, zfs_backup will continue and report the number of
failures at that end. Also the exit code will indicate the number of failures.
```

Backup example
==============

In this example we're going to backup a SmartOS machine called `smartos01` to our fileserver called `fs1`.

Its important to choose a unique and consistent backup name. In this case we name our backup: `smartos01_fs1`.

Select filesystems to backup
----------------------------

On the source zfs system set the ```autobackup:smartos01_fs1``` zfs property to true:
```
[root@smartos01 ~]# zfs set autobackup:smartos01_fs1=true zones
[root@smartos01 ~]# zfs get -t filesystem autobackup:smartos01_fs1
NAME                                                PROPERTY                  VALUE                     SOURCE
zones                                               autobackup:smartos01_fs1  true                      local
zones/1eb33958-72c1-11e4-af42-ff0790f603dd          autobackup:smartos01_fs1  true                      inherited from zones
zones/3c71a6cd-6857-407c-880c-09225ce4208e          autobackup:smartos01_fs1  true                      inherited from zones
zones/3c905e49-81c0-4a5a-91c3-fc7996f97d47          autobackup:smartos01_fs1  true                      inherited from zones
...
```

Because we dont want to backup everything, we can exclude certain filesystem by setting the property to false:
```
[root@smartos01 ~]# zfs set autobackup:smartos01_fs1=false zones/backup
[root@smartos01 ~]# zfs get -t filesystem autobackup:smartos01_fs1
NAME                                                PROPERTY                  VALUE                     SOURCE
zones                                               autobackup:smartos01_fs1  true                      local
zones/1eb33958-72c1-11e4-af42-ff0790f603dd          autobackup:smartos01_fs1  true                      inherited from zones
...
zones/backup                                        autobackup:smartos01_fs1  false                     local
zones/backup/fs1                                    autobackup:smartos01_fs1  false                     inherited from zones/backup
...
```


Running zfs_autobackup
----------------------
There are 2 ways to run the backup, but the endresult is always the same. Its just a matter of security (trust relations between the servers) and preference.

First install the ssh-key on the server that you specify with --ssh-source or --ssh-target.

Method 1: Run the script on the backup server and pull the data from the server specfied by --ssh-source. This is usually the preferred way and prevents a hacked server from accesing the backup-data:
```
root@fs1:/home/psy# ./zfs_autobackup --ssh-source root@1.2.3.4 smartos01_fs1 fs1/zones/backup/zfsbackups/smartos01.server.com --verbose
Getting selected source filesystems for backup smartos01_fs1 on root@1.2.3.4
Selected: zones (direct selection)
Selected: zones/1eb33958-72c1-11e4-af42-ff0790f603dd (inherited selection)
Selected: zones/325dbc5e-2b90-11e3-8a3e-bfdcb1582a8d (inherited selection)
...
Ignoring: zones/backup (disabled)
Ignoring: zones/backup/fs1 (disabled)
...
Creating source snapshot smartos01_fs1-20151030203738 on root@1.2.3.4
Getting source snapshot-list from root@1.2.3.4
Getting target snapshot-list from local
Tranferring zones incremental backup between snapshots smartos01_fs1-20151030175345...smartos01_fs1-20151030203738
...
received 1.09MB stream in 1 seconds (1.09MB/sec)
Destroying old snapshots on source
Destroying old snapshots on target
All done
```

Method 2: Run the script on the server and push the data to the backup server specified by --ssh-target:
```
./zfs_autobackup --ssh-target root@2.2.2.2 smartos01_fs1 fs1/zones/backup/zfsbackups/smartos01.server.com --verbose  --compress
...
All done

```

Tips
====

 * Set the ```readonly``` property of the target filesystem to ```on```. This prevents changes on the target side. If there are changes the next backup will fail and will require a zfs rollback. (by using the --rollback option for example)
 * Use ```--clear-refreservation``` to save space on your backup server.
 * Use ```--clear-mountpoint``` to prevent the target server from mounting the backupped filesystem in the wrong place during a reboot. If this happens on systems like SmartOS or Openindia, svc://filesystem/local wont be able to mount some stuff and you need to resolve these issues on the console.

Speeding up SSH and prevent connection flooding
-----------------------------------------------

Add this to your ~/.ssh/config:
```
Host *
    ControlPath ~/.ssh/control-master-%r@%h:%p
    ControlMaster auto
    ControlPersist 3600
```

This will make all your ssh connections persistent and greatly speed up zfs_autobackup for jobs with short intervals.

Thanks @mariusvw :)


Specifying ssh port or options
------------------------------

The correct way to do this is by creating ~/.ssh/config:
```
Host smartos04
    Hostname 1.2.3.4
    Port 1234
    user root
    Compression yes
```

This way you can just specify "smartos04" as host.

Also uses compression on slow links.

Look in man ssh_config for many more options.

Troubleshooting
===============

`cannot receive incremental stream: invalid backup stream`

This usually means you've created a new snapshot on the target side during a backup.
 * Solution 1: Restart zfs_autobackup and make sure you dont use --resume. If you did use --resume, be sure to "abort" the recveive on the target side with zfs recv -A.
 * Solution 2: Destroy the newly created snapshot and restart zfs_autobackup.


`internal error: Invalid argument`

In some cases (Linux -> FreeBSD) this means certain properties are not fully supported on the target system.

Try using something like: --filter-properties xattr


Restore example
===============

Restoring can be done with simple zfs commands. For example, use this to restore a specific SmartOS disk image to a temporary restore location:


```
root@fs1:/home/psy#  zfs send fs1/zones/backup/zfsbackups/smartos01.server.com/zones/a3abd6c8-24c6-4125-9e35-192e2eca5908-disk0@smartos01_fs1-20160110000003 | ssh root@2.2.2.2 "zfs recv zones/restore"
```

After that you can rename the disk image from the temporary location to the location of a new SmartOS machine you've created.


Monitoring with Zabbix-jobs
===========================

You can monitor backups by using my zabbix-jobs script. (https://github.com/psy0rz/stuff/tree/master/zabbix-jobs)

Put this command directly after the zfs_backup command in your cronjob:
```
zabbix-job-status backup_smartos01_fs1 daily $?
```

This will update the zabbix server with the exitcode and will also alert you if the job didnt run for more than 2 days.


Backuping up a proxmox cluster with HA replication
==================================================

Due to the nature of proxmox we had to make a few enhancements to zfs_autobackup. This will probably also benefit other systems that use their own replication in combination with zfs_autobackup.

All data under rpool/data can be on multiple nodes of the cluster. The naming of those filesystem is unique over the whole cluster. Because of this we should backup rpool/data of all nodes to the same destination. This way we wont have duplicate backups of the filesystems that are replicated. Because of various options, you can even migrate hosts and zfs_autobackup will be fine. (and it will get the next backup from the new node automaticly)


In the example below we have 3 nodes, named h4, h5 and h6.

The backup will go to a machine named smartos03.

Preparing the proxmox nodes
---------------------------

On each node select the filesystems as following:
```
root@h4:~# zfs set autobackup:h4_smartos03=true rpool
root@h4:~# zfs set autobackup:h4_smartos03=false rpool/data
root@h4:~# zfs set autobackup:data_smartos03=child rpool/data

```

* rpool will be backuped the usual way, and is named h4_smartos03. (each node will have a unique name)
* rpool/data will be excluded from the usual backup
* The CHILDREN of rpool/data be selected for a cluster wide backup named data_smartos03. (each node uses the same backup name)


Preparing the backup server
---------------------------

Extra options needed for proxmox with HA:
* --no-holds: To allow proxmox to destroy our snapshots if a VM migrates to another node.
* --ignore-replicated: To ignore the replicated filesystems of proxmox on the receiving proxmox nodes. (e.g: only backup from the node where the VM is active)


I use the following backup script on the backup server:
```
for H in h4 h5 h6; do
  echo "################################### DATA $H"
  #backup data filesystems to a common place
  ./zfs_autobackup --ssh-source root@$H data_smartos03 zones/backup/zfsbackups/pxe1_data --clear-refreservation --clear-mountpoint  --ignore-transfer-errors --strip-path 2 --verbose --resume --ignore-replicated --no-holds $@
  zabbix-job-status backup_$H""_data_smartos03 daily $? >/dev/null 2>/dev/null

  echo "################################### RPOOL $H"
  #backup rpool to own place
  ./zfs_autobackup --ssh-source root@$H $H""_smartos03 zones/backup/zfsbackups/$H --verbose --clear-refreservation --clear-mountpoint  --resume --ignore-transfer-errors $@
  zabbix-job-status backup_$H""_smartos03 daily $? >/dev/null 2>/dev/null
done
```
