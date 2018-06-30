## Introduction

Rocks is basically a set of Python programs that controlls and facilitates cluster management. For user account management, once you make changes in the head node, there is a specific command `rocks sync users` that propagates all user information across all compute nodes. In recent versions of Rocks, now host-based authentification is used, so that any user can log into a compute node without typing password (previously, SSH key-based authentification is used, but occasionally it has problems for various reasons).

## Setting up home directory

For the setup of my cluster, the only place that I need to change Rocks source code is for user account administration. The reason is that I wanted to use a common storage server called `nas-0-0`, rather than head node, to store all user files, yet Rocks does not have an easy way to do this, unless I change source code. Fortunately, it is written in Python, so changing them is straightforward.

Preparation: In nas-0-0, add `/dev/mapper/nas--0--0-export    /export         xfs     defaults        0 0` to the `/etc/fstab` file. Then mount the directory `mount /export`. Edit `/etc/export`, and add these two lines:

```
/export 10.1.0.0/255.255.0.0(rw,no_root_squash,async)
/export 192.168.0.0/255.255.0.0(rw,no_root_squash,async)
```

Then execute `exportfs -r` (or use -a) to make it effective immediately. Then create `/export/home` directory in nas-0-0. We can now set up the user home directory below.

First, do `rocks set attr Info_HomeDirSrv nas-0-0-ib.ipoib` to set up the server address for home directory. Then in `/opt/rocks/lib/python2.6/site-packages/rocks/commands/sync/users/plugin_fixnewusers.py`, just change value of “default_dir” from "/export/home/" to “/mnt/nas-0-0/home” (but make sure to first create this directory in head node). Next, change `/etc/default/useradd`, and use `HOME=/mnt/nas-0-0/home`. Finally, edit `/etc/auto.nas` and add `home    nas-0-0-ib.ipoib:/export/home` into the file, and edit `/etc/auto.master` and add `/mnt/nas-0-0    /etc/auto.nas   --timeout=1200` into the file. These are the necessary changes to set up user home directory correctly.

Note that by default, the "/export/home" was used in useradd, but we cannot do automount for "/export" as the directory contains several sub-directories including home. So we have to use a new directory called /mnt/nas-0-0/home to do this. So modifying the source code is required to make this work.

Under the scene, this is what happened when I add a new user called `kaiwang5`:

1. when typing `useradd kaiwang5`, the `/etc/password` file is changed and a new directory `/mnt/nas-0-0/kaiwang5` is created (which depends on setting in `/etc/default/useradd` that we changed above). It is important to note that `/mnt/nas-0-0` is automounted by autofs (based on the `/etc/auto.master` file that we edited above), and in fact, the only time it is needed is when people use `useradd`.
2. when typing `rocks sync users`, the `/etc/password` file is changed from `/mnt/nas-0-0/kaiwang5` to `/home/kaiwang5`, and the `/etc/auto.home` is appended with the correct mounting points for home directory (the parameter are determined by `Info_HomeDirSrv` and `Info_HomeDirLoc` system parameters, or if not set, determined by the  `/opt/rocks/lib/python2.6/site-packages/rocks/commands/sync/users/plugin_fixnewusers.py` command).
3. You can perhaps now understand why we need to modify `/opt/rocks/lib/python2.6/site-packages/rocks/commands/sync/users/plugin_fixnewusers.py` and change “default_dir”. 
4. we should automount each user, not the whole `/home`. Therefore, autofs is used for every user, as specified in the `/etc/auto.home` file.
5. `rocks list attr` will print out all Rocks variables. You may want to examine them and change some of them. For example, the `Info_HomeDirSrv` needs to be `nas-0-0-ib.ipoib`, rather than `nas-0-0`, to make sure that we use IB traffic for NFS.

Additional update in 2018: I tested the procedure above and it works well for the new Rocks 7. The only difference is to change python2.6 to python2.7 above. Another slight difference relates to IB configuration, where one has to manually add IB port, as it is not automatic (all the ethernet ports are automatically detected though).

## Standard procedure for adding a new user

1. login by “su -”
2. useradd -c “Kai Wang” -g wanglab kaiwang
3. passwd kaiwang
4. rocks sync users
5. log in, make sure that everything is fine

the current gid and uid information for all users can be accessed in the `/etc/passwd` file.

> Troubleshooting: once I add the new account panda, the home directory cannot be accessed, and the `/etc/passwd` file shows `panda:x:516:518:Pandiyan:/mnt/nas-0-0/home/panda:/bin/bash`. I suspect that there is an issue with automount, so I did `service restart autofs`, then do `usermod -d /home/panda panda`, and then everything returns normal.

## Useful commands for account changes

- add a group
```
[root@biocluster ~]# groupadd wanglab
```

- delete account
```
[root@biocluster ~]# userdel -r kaiwang5
```

Note that the `-r` will delete this user cleanly; otherwise mailbox and other things may still exist)
```
[root@biocluster ~]# rocks sync users
```
 
- change a user group (only after first log in)

`usermod -g wanglab kaiwang5`
 
- change user home directory to group home (after changing user's group, we need to change the files to this group)

`chgrp -c -R wanglab /home/kaiwang5`

## disable and lock a user

```
usermod -L -e 1 guest
```

The `-L` option lock user's password by putting a ! in from of the the encrypted password. (The `passwd -l guest` can do similar thing). To disable user account set expire date to 1.

Note that locking password per se will not prevent a user from logging in from ssh key authentification. So setting expiration date is important.

```
chage -E 0 guest
```

will do a full account locking for guest.

Use chage command to see current status of the user account: `chage -l guest`

Use this to re-enable the user account: `chage -E -1 guest`. Use this to unlock the user account: `usermod -U guest`.

### UID and GID

To find a user's UID or GID in Unix, use the `id` command. To find a specific user's UID, at the Unix prompt, enter: `id -u username`
 
Replace username with the appropriate user's username. To find a user's GID, at the Unix prompt, enter: `id -g username`
 
If you wish to find out all the groups a user belongs to, instead enter: `id -G username`
 
If you wish to see the UID and all groups associated with a user, enter id without any options, as follows: `id username`

### Kill all user process

```
 echo `ps -fu $User | awk 'NR != 1 {print $2}'`
```

Once you're sure that you're eching the right stuff, replace the echo with kill.

Alternatively, just do a `killall -u <user>`.
