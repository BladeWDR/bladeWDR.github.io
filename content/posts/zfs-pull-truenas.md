+++
title = "How I set up ZFS replication backups in TrueNAS Scale"
date = "2024-02-10T14:58:56-05:00"
author = "Blade"
authorTwitter = "BladeWDR" #do not include @
cover = ""
tags = ["zfs", "linux", "truenas"]
keywords = ["zfs", "linux", "truenas"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

## How I set up ZFS replication backups in TrueNAS Scale

When I set out to configure my ZFS replication backups, I knew that I wanted 2 things:

1. I wanted to do my replications with an unprivileged user.
2. I wanted the backups to be *pulled* from the backup server from the primary.

The benefits of using a user without root privileges are obvious, but the benefits of a pull backup may be less so.

Backing up with a pull instead of a push means that your production machine, which by definition has to be less secure than the backup, has no access to the backup server.

So, if somehow your production machine gets compromised - the attacker has no way of accessing the backup server, and possibly taking those hostage too!

Of course, you should make sure that you take steps to lock down your backup server.

* Make sure that SSH login only accepts public key authentication, instead of passwords.
* Goes without saying that your production machine *should not have the the private key for this*.
* Set up firewall rules to prevent anything from connecting to the machine that isn't supposed to. You probably want to poke holes for DNS resolution, and let the server get out to the internet for updates, but apart from that and a hole for your monitoring solution, the backup server should be totally isolated from everything else on your network!

#### Step 1: Create your user accounts.

You need to create a user on each side. On the *sending* side, or in other words, your production server, you'll need to add an SSH authorized key, and enable a home directory.
I recommend disabling password login entirely, and disabling samba authentication. You won't be using this user for anything but the backups.

Create another user on the backup server. I recommend matching usernames, just for clarity, but it's not 100% necessary.

![user creation panel](/truenasuser.png)

#### Step 2: Assign privileges.

In order to let this user account run zfs commands without root, you'll need to drop to the shell. There are options in the TrueNAS web UI for allowing certain sudo commands without a password, but it's better to do this on the command line, as it gives you more control over what specific rights your user has.

On the receiving side, your user needs the receive, create, and mount privileges.

`zfs allow pirate receive,create,mount pool`

On the sending side, your user needs the send and hold privileges.

`zfs allow pirate send,hold pool`


#### Step 3: Create your backup credentials in TrueNAS.

Head back to the TrueNAS web UI and click on Credentials ---> Backup Credentials. You want to add a new SSH connection.

Choose manual as your setup method.

Fill in as below.

![backup credentials](/truenas_ssh.png)

#### Step 4: Create your ZFS replication task.

In the TrueNAS web UI, go to Data Protection and expand Replication Tasks.

You'll want to choose your "SSH Connection" first. Pick the one that you created in Step 3.

Next, choose your source location. You'll want to pick "On a different system", then choose the dataset or pool you want to replicate below.

Do the same for your destination, which should be on a local ZFS pool.

If you want to copy child datasets, make sure you check off "Recursive". In my case, this dataset doesn't have any children, so I left it unchecked.

If at any point you get the below prompt - make sure you click "Cancel". You'll need to hit Cancel twice. We don't need sudo, since we manually allowed our user account to run the needed zfs commands using `zfs allow`.

![sudo prompt](/sudo_cancel.png)

Click Next when you're ready. Choose a schedule. I chose "Daily", as it's not a big deal for this dataset if I lose a single day.

And there you have it. You now have a much more secure method of replicating your data from one ZFS pool to another.

![zfs replication wizard](/replication_wizard.png)

#### Some additional notes.

* Remember that you'll need to have some snapshots on the source system for this to work. ZFS replication works based off of *snapshots*, because they are a consistent, immutable reference at a point in time. I recommend setting up automatic snapshot tasks for the datasets you want to back up.
* Once you've completed the initial replication job, any subsequent jobs will be incrementals - as long as you have a common snapshot between the two hosts.
