# How to Set Up an NFS Server on Ubuntu (Complete with AutoFS!)

## NFS-Server
Create Base Directory/Parent directory for all of our NFS shares. What we are going to do is to create some sub directories within that parent directory and each of those subdirectories will be NFS shares in and of themselves.

So lets create the first directory now.
```shell
sudo mkdir /exports
```

```shell
cd /exports
#check files list, it's empty now
ls
# In there, Create couple other Directories now. random directories
sudo mkdir backup

sudo mkdir documents
# List files and directories again
ls

```
Having one Parent Directory is not actually required, but it is a good practice. Now we have those particular directories, thatwe plan on sharing, we need a means to share those directories, that's where NFS comes in.

But the NFS server Service is not actually installed right now, let's take care of that.

### Install NFS Server
```shell
sudo apt install nfs-kernel-server
```
As the NFS package installed, now we have created the real nfs server, because we have an nsf service running. But we still have to do more setup!

The default configuration does not know about directories we plan on sharing.
let's go ahead and st that up.

To make sure everything is okay, lets check the status of the nfs server.
```shell
systemctl status nfs-kernel-server
```
```shell
: '
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset:
     Active: active (exited) since Thu 2023-08-31 07:40:40 UTC; 8min ago
   Main PID: 13660 (code=exited, status=0/SUCCESS)
        CPU: 4ms

Aug 31 07:40:39 nfs-server-0 systemd[1]: Starting NFS server and services...
Aug 31 07:40:39 nfs-server-0 exportfs[13658]: exportfs: can't open /etc/exports for reading.
Aug 31 07:40:40 nfs-server-0 systemd[1]: Finished NFS server and services.
 '
```
At this moment it is active but exited because nothing for it to share. We haven't configure anything for it to do. So it started, it looked at the ```/etc/exports``` file and that is actualy the text file that will list all the directories inside of what we plan on sharing, but because we haven't actually
added anything yet, there is nothing for the NFS Server to do, so the service has exited.

Lets configure that file right now
```shell
# Let's check that file before doing anything with it
cat /etc/exports
# Current File Output
: '
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)

'
# Keep this original file for future reference, just in case.
sudo mv /etc/exports /etc/exports.original
#Create a fresh one
sudo nano /etc/exports
#paste these two lines, add network identifier and the subnet
# /exports/backup 192.168.10.0/255.255.255.0(rw,no_subtree_check)
# /exports/documents 192.168.10.0/255.255.255.0(rw,no_subtree_check)
'
```
The subnet you add determines which host can actually connect to this nfs server and access the directories that you share.

You could make it wide open by not including the IP address or subnet at all, I do not recommend it, but it is your server. What I do recommend you todo, is to find out what your IP subnet actually is. `You can just grab your IP address from any host on your network and change the last octet to zero`.

And that should actually get you where you need to be. I f you do encout problems, just remove the network and the subnet identifier like I have mentioned, but again I recommend you keep those in there, and you just configure this to match your particular environment.

After the subnet we see that we have some options in parenthesis. Among those we have `RW` which you could probably guess means `Read-Write`, You could change it to only allow for Read-Only if you not want any changes to be made to the contents inside those directories.

And we also have the option `no_subtree_check`. This option disable the parent directory of the export for being apart of the file handle. Basicaly waht we are doing, is that we are disabling subtree checking  completely in favor of security because threat actors can actually use the parent directory information against us. long story short, it helps with security a bit.

Save the file, and believe or not, we have actually finished setting up the NFS server.

>**Note**: The Changes we have made will not take effect untill we restart our NFS server.

Restart the Server
```shell
sudo systemctl restart nfs-kernel-server
```

Now Check the status, We hope it is not exited as it was last time.
```shell
sudo systemctl status nfs-kernel-server
: '
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset: e>
     Active: active (exited) since Thu 2023-08-31 08:42:30 UTC; 2min 4s ago
    Process: 35833 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Process: 35834 ExecStart=/usr/sbin/rpc.nfsd (code=exited, status=0/SUCCESS)
   Main PID: 35834 (code=exited, status=0/SUCCESS)
        CPU: 4ms

Aug 31 08:42:30 nfs-server-0 systemd[1]: Starting NFS server and services...
Aug 31 08:42:30 nfs-server-0 systemd[1]: Finished NFS server and services.

'
```
Now It is no longer complaining that `/etc/exports` file does not contain anything useful! It looks like everythis was a Success.

#### Prove that its working!

What we are going to do now is to create some text file to prove that this is working. I change the directory into the parent directory for nfs if we are not already there.

```shell
cd /exports

sudo nano  backup/text1.txt
#paste this content and save
# Hello World
sudo nano documents/text2.txt
#paste this content and save
# Hello from Shared Documents
```

We are all set to the NFS-server

## NFS-Client
Lets go ahead and see how the process looks like when connection to the shared directories.

The first thing we are going to do is to install NFS Client package.

```shell
# That is the name of the package, it contains the utilities that we need to connect to an NFS share from another server.
sudo apt install nfs-common
```
That it is it when it comes to an NFS client.
I recooment that you test connectivity to the server ans there is actually a dedicated command that you can you for that purpose. and that command is the `showmount` command.

We could use thisncommand to connect to the server and see which directories are being shared by NFS. but you wiil need the IP address of the server if you don't happen to know this already.

To find that out, Jusr go back to the NFS server terminal and run:

```shell
ip addr show
#or
ip a show
```
And mine is `192.168.10.10`. and now you know the IP address of the server, we can continue.

Back to the Client terminal
```shell
#showmount --exports Server-IP-Address
showmount --exports 192.168.10.10
# Output
: '
Export list for 192.168.10.10:
/exports/documents 192.168.10.0/255.255.255.0
/exports/backup    192.168.10.0/255.255.255.0
'
```
Success! it shows both of directories right there that we have shared from the NFS server and that can only mean that the NFS client has no problem at all when it comes to connecting the the NFS server.

We should be good to go to proceed.

Similarly how we created a Parent directory on the server to act as a staring point for any directories that we plan on sharing, we are going to do similar thing here on the client. `We are going to create a Parent Directory` that are going to include `subdirectories`, and each on of `those subdirectories will be attached to one of those shared directories from theserver`

```shell
sudo mkdir /mnt/nfs

```
The `/mnt` already existed. We have that on our file system, we diid not need to create that.
But we are going to use the `nfs` directory underneath that directory to mount our dirctories underneath.

Now what we do is the create subdirectories and each of these will be named the same as the directory that's being shared. We just want to be consistent but you do not have to keep the names the same, But it is a good Idea anyway.

```shell
sudo mkdir /mnt/nfs/backup
```
```shell
sudo mkdir /mnt/nfs/documents
```
Check our recently created directories

```shell
ls -l /mnt/nfs
: '
total 8
drwxr-xr-x 2 root root 4096 Aug 31 09:55 backup
drwxr-xr-x 2 root root 4096 Aug 31 09:56 documents
'
```
Check our subdirectories
```shell
ls -l /mnt/nfs/backup
# These directories are empty
: '
total 0
'
```
These directories are empty just because we have not mounted anything yet.

### Mount na NFS Export
Let's go ahead and see how we go about mounting an NFS export.

```shell
sudo mount 192.168.10.10:/exports/backup /mnt/nfs/backup
```
Check disk usage to test our mounted share.
```shell
df -h
: '
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              198M  1.1M  197M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.8G   26G  13% /
tmpfs                              988M     0  988M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
vagrant                            931G  415G  516G  45% /vagrant
tmpfs                              198M  4.0K  198M   1% /run/user/1000
192.168.10.10:/exports/backup       31G  3.8G   26G  13% /mnt/nfs/backup
'
```
Take a closer look at the end, you see that our shared `/exports/backup` subsirectory `from the server` is mounted on the `mnt/nfs/backup` on the client.

Verify!
```shell
# Remember we created the text1.tx on the nfs-server in the backup shared subdirectory.
cat /mnt/nfs/backup/text1.txt
# Output
: '
Hello World
'
```
Now we definetely know NFS is working.

Let's go ahead and mount other directory.
```shell
sudo mount 192.168.10.10:/exports/documents /mnt/nfs/documents
```
Check disks usage
```shell
df -h
: '
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              198M  1.1M  197M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.8G   26G  13% /
tmpfs                              988M     0  988M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
vagrant                            931G  415G  516G  45% /vagrant
tmpfs                              198M  4.0K  198M   1% /run/user/1000
192.168.10.10:/exports/backup       31G  3.8G   26G  13% /mnt/nfs/backup
192.168.10.10:/exports/documents    31G  3.8G   26G  13% /mnt/nfs/documents

'
```
Again Take a closer look at the end, you see that our shared `/exports/documents` subsirectory `from the server` is mounted on the `mnt/nfs/documents` on the client.

The `documents` Exports has been now mounted.

Check files in there
```shell
ls -l /mnt/nfs/documents
: '
total 4
-rw-r--r-- 1 root root 28 Aug 31 09:09 text2.txt
'
```
Verify the file
```shell
cat /mnt/nfs/documents/text2.txt
: '
Hello from Shared Documents
'
```

### Unmount NFS Exports

How do you unmount NFS exports when you arefinished with them and you no longer need them?

Unmount `backup` directory
```shell
sudo umount /mnt/nfs/backup
```
Unmount `documents` directory
```shell
sudo umount /mnt/nfs/documents
```
Check that!
```shell
df -h
#Output
: '
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              198M  1.1M  197M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.8G   26G  13% /
tmpfs                              988M     0  988M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
vagrant                            931G  416G  515G  45% /vagrant
tmpfs                              198M  4.0K  198M   1% /run/user/1000
'
```
As you can see, `backup` and `documents` exports are no longer mounted.


### On-demand NFS mounting with AutoFS
How do we get these particular maounts get automatically mounted? that is where AutoFS comes handy. It allows you have these mounted on demand.

And you might be wondering why does that matter? Well NFS does legitimately have some locking issues that sometimes creep up. So by using the on-demand system, I find that it works better and the beauty of autoFS is that it mounts things so quickly that any app that might be expecting to find that those directories mounted, won't even have any idea that they ever weren't mounted.

So lets go ahead and see how we setup AutoFS to automatically Mount these exports.

First of all what I do in to navigate to `/mnt/nfs` where we have thse subdirectories on the NFS-client, Remove those sub directories because AutoFS takes care those for us.

Bofore We actually go about removing anything, we will definetely make sure that, and double check that nothing is mounted. Beacause if we run RM command against something that is mounted, we might risk loosing the contents of that particular exports. 

And we definetely don't want that, expecially if we have something important saved inside that directory.

Check if there is any mounted export
```shell
df -h
: '
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              198M  1.1M  197M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.8G   26G  13% /
tmpfs                              988M     0  988M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
vagrant                            931G  416G  515G  45% /vagrant
tmpfs                              198M  4.0K  198M   1% /run/user/1000
'
```
None of them is mounted. We are good to go and delete subdirectories exports.

```shell
cd /mnt/nfs
#list files & directories
ls
: '
backup  documents
'
```
Remove these directories as AutoFs will be in charge to create them automatically.

```shell
# Inside /mnt/nfs/
sudo rm -r backup documents
# Check if deleted successfully
ls
```

#### Install AutoFS Package
Now Install AutoFS package
```shell
sudo apt-get update

sudo apt install autofs
```

Check AutoFS service status

```shell
sudo systemctl status autofs
: '
● autofs.service - Automounts filesystems on demand
     Loaded: loaded (/lib/systemd/system/autofs.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2023-08-31 12:07:50 UTC; 2min 55s ago
       Docs: man:autofs(8)
    Process: 64365 ExecStart=/usr/sbin/automount $OPTIONS --pid-file /var/run/autofs.pid (code=exited, status=0/SUCCESS)
   Main PID: 64367 (automount)
      Tasks: 3 (limit: 2233)
     Memory: 1.3M
        CPU: 5ms
     CGroup: /system.slice/autofs.service
             └─64367 /usr/sbin/automount --pid-file /var/run/autofs.pid

Aug 31 12:07:50 nfs-client-0 systemd[1]: Starting Automounts filesystems on demand...
Aug 31 12:07:50 nfs-client-0 systemd[1]: Started Automounts filesystems on demand.
vagrant@nfs-client-0:/mnt/nfs$

'
```

That is Cool, but we still do not have anything mounted yet.
```shell
df -h
: '
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              198M  1.1M  197M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.8G   26G  13% /
tmpfs                              988M     0  988M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
vagrant                            931G  416G  515G  45% /vagrant
tmpfs                              198M  4.0K  198M   1% /run/user/1000

'
```
That is because AutoFS actually needs to be configured.

- In fact, there is two config files we need to edit to make this work.

1. Tha first one is `/etc/auto.master` file.
```shell
sudo nano /etc/auto.master
# Paste this content at the very end of the file
: '
/mnt/nfs /etc/auto.nfs --ghost --timeout=60
'
```
- The `/mnt/nfs` : Is the exact same directory that we created as our base directory for mounting NFS exports. definetely makes sure this matches.

- The `/etc/auto.nfs` : this the file that is already exist on the file system that autofs created for us when autofs itself was installed.

So what we aere doing right here by adding this particular path to this file is that we are telling AutoFS that it looks at this file for the  different mounts that we want to be a part of the NFS config that we have here.

And anything that is included in the `auto.nfs` file is going to be mounted underneath the `/mnt/nfs` directory.

- The Option `--ghost` : Creates ghost directories that will exist inside parent directory and it will make sure that those directories already exist inside that Directory.

That is imprortant because if a program or a process goes to look for any of these directories  that are suppored to be mounted, it at least needes to find the subdirectory and if it does not find it, it could well fail. We do not want that!

So the Gost option will make sure that those directories exist.

- The `--timeout` : The timeout that is set to 60 seconds, waht it does is that instructs AutoFS to unmount any NFS shares that that haven't been accessed or touched for that number of seconds and that actually adds to its dynamic nature.

So Its a good option to have `especially if you have a server that goes offline and the client is always going to wait for 60 seconds before it gives up`, otherwise it could try forever and then well freeze.

You definitely do not want that.


2. Let's move on to the next file, `/etc/auto.nfs`.
```shell
sudo nano /etc/auto.nfs
# Paste the following lines
: '
backup -fstype=nfs4,rw 192.168.10.10:/exports/backup
documents -fstype=nfs4,rw 192.168.10.10:/exports/documents
'
```

Asyou can here here we have two mounts identified that I want autoFS to keep track of. and each of these lines corresponds to one of the exports that we created on the server.

- The first line: Is actually for the backup export. So at the very beginning  we are actually naming this backup. But this might be a bit confusing at first, `so its important to understand` that `backup` right here `does not correspond to the name of the directory on the server`, but instead it corresponds to waht we want the directory to be called on the client, when its mounted.

And this directory in this case `backup`, will be created underneath the parent directory that `we have identified in the /etc/auto.master` file

So the name does not technically have to match, but i recommend that you make it match because it makes sense. Just like averything to as consistent as possibly can be.

- But to be fair where you see backup right here you could have called this something else and it would be fine. Whatever you name it right here is what `the subdirectory or the Ghost directory` is going to be named inside the parent directory.

- Next we are setting the file system type, its `nfs4` in this case, then we have a coma followed by `rw` for read write. So on the server you can set `read-write` or `read-only`, But on the client you can also do the same.

- next we have an IP address that goes to the server tht contains the directories that we wish to mount. Then we have a Colon `:` and then the full path to the directory on the server where the actual exports is located.

So you recall we created our shares under the slash exports directory, that is what we have right here, and the directory itself that we are sharing out. That's really all there is to it.

- To understsn how these 2 files relate to each other, See relavent lines of both configs at the same time:
```shell
tail -n 1 /etc/auto.master
#Output
: '
/mnt/nfs /etc/auto.nfs --ghost --timeout=60
'
cat /etc/auto.nfs
#Output
: '
backup -ftype=nfs4,rw 192.168.10.10:/exports/backup
documents -ftype=nfs4,rw 192.168.10.10:/exports/documents
'
```

- autto.master file is the very first files that autoFS reads when it starts up. When it reads that file what is going to find is this line `/mnt/nfs /etc/auto.nfs --ghost --timeout=60` right here.

- On `backup -ftype=nfs4,rw 192.168.10.10:/exports/backup, documents -ftype=nfs4,rw 192.168.10.10:/exports/documents` lines backup and document will be mounted into `mn/nfs` that is specified in the first file.

- That is how the two files matches together.

Lets Restart AutoFS as we have all configuratiions together.

For cquick sanity check
```shell 
df -h 
: '
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              198M  1.1M  197M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.8G   26G  13% /
tmpfs                              988M     0  988M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
vagrant                            931G  416G  515G  45% /vagrant
tmpfs                              198M  4.0K  198M   1% /run/user/1000
'
```
Nothing is mounted. That is waht we expect

- Restart AutoFS
```shell
sudo systemctl restart autofs
```
- Let’s run df -h to see what we have mounted now (they shouldn’t be, at least not yet):
```shell
df -h
: '
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              198M  1.1M  197M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.8G   26G  13% /
tmpfs                              988M     0  988M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
vagrant                            931G  416G  515G  45% /vagrant
tmpfs                              198M  4.0K  198M   1% /run/user/1000
'
```
The way that AutoFS works, your NFS shares are not mounted until a user or process attempts to access them. When an application attempts to even so much as list the contents of one of those directories, it will automatically trigger the directory to be mounted on the spot. It’ll happen so fast, that the app will never be aware that it was ever not mounted.

- To test this, list the contents of the parent directory or one of the exports:
```shell
ls -l /mnt/nfs
```
- Since listing the contents of even the base directory requires information to be displayed from the exports, we should see that this triggers a mount to happen:

```shell
df -h
: '
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              198M  1.1M  197M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.8G   25G  14% /
tmpfs                              988M     0  988M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
vagrant                            931G  416G  515G  45% /vagrant
tmpfs                              198M  4.0K  198M   1% /run/user/1000
192.168.10.10:/exports/backup       31G  3.8G   26G  13% /mnt/nfs/backup
192.168.10.10:/exports/documents    31G  3.8G   26G  13% /mnt/nfs/documents
'
```
Once the shares become idle, they will automatically unmount. Once you go to access them, AutoFS will mount the directories again.

### Thank You!

