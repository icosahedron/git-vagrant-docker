# Introduction #

This is a simple configuration for Vagrant and Docker that sets up a [Gogs](https://gogs.io) Git server with an nginx reverse proxy for https support.  It shares the SSL certificates and the git repos from the host to make backups easier.

Using Vagrant gives a consistent machine set up for the Docker containers, without interfering the host machine.  The configuration works best on macOS, Linux, or other Unix OS's.  Windows support is forthcoming.

# Prerequisites #

There are a few requirements that must be installed prior to running the Git server.

## NFS Setup ##

To make backups and configuration easier, Vagrant shares the git repos and SSL certs with the host via NFS[^nfs-shares].  Vagrant takes care of the exports, as long as NFS is installed.  If you're running macOS[^nfs-instructions-mac] or Linux, it should already be installed, or can easily be with a trip to your package manager.

[^nfs-instructions-mac]: I used these [instructions](http://ilgthegeek.old.livius.net/2015/04/18/os-x-start-nfs-daemon/) to enable NFS on macOS

[^nfs-shares]: We use NFS shares rather than the VirtualBox shared folders due to [performance concerns](https://www.jeffgeerling.com/blogs/jeff-geerling/nfs-rsync-and-shared-folder).

If you're on macOS, I recommend checking your NFS configuration by downloading [NFS Manager](https://www.bresink.com/osx/143439/download.php).  It makes it much easier than trying to deal with configuration files.

## Become Certified ##

To provide SSL, nginx is configured to look for [certificates from Let's Encrypt.](https://letsencrypt.org)  Vagrant and Docker are configured to look for the certs in `/etc/letsencrypt` on the host via NFS.

## Not All Who Wander Are Lost ##

Install [Vagrant](https://www.vagrantup.com) via your favorite package manager, or [download it](https://www.vagrantup.com/downloads.html) directly.  I use the VirtualBox provider as it is free and adequate for my purposes.

## Pull the Trigger ##

I use vagrant-triggers to install Docker Compose and bring up the containers.  Install the [`vagrant-triggers`](https://github.com/emyl/vagrant-triggers) with

```
$ vagrant plugin install vagrant-triggers
```

# Installation and Configuration #

Installation should be as easy as cloning this repo.  The repo should be placed where you want the VM to run, or it can be copied to where you want the VM to run.

The Vagrant `config.rb` file contains the configuration options you might be interested in changing:

* $instance-name-prefix - set to "git-server", this is a simple placeholder if you care for something better.  Doesn't affect too much.
* $vm_memory and $vm_cpus - amount of memory and number of host CPUs dedicated to the Vagrant VM.
* $nfs_shared_folders - the folders meant to be shared.  If you have installed your certs somewhere else, make the change here.  Also, this configuration is set up to place the git repos under the gogs directory in the repo.  If you wish those to be placed somewhere else, map that here.
* $forwarded_ports - map ports on the Vagrant guest ports to the ports on the host.

The `Vagrantfile` itself has one notable configuration option.  Where the the NFS mounts are created, I set `nfs_udp: false` since my NFS mounts wouldn't work with UDP.  However, this may affect performance, so you might want to enable UDP to see if NFS works with it in your case.  Note, there is a good [reason](ttps://github.com/mitchellh/vagrant/issues/2304) to keep TCP enabled even for local NFS mounts.

The `docker-compose.yml` in docker subdirectory shouldn't need any changes, though the host and domain names should be changed to your own site.

Once complete, you can simply bring up the Vagrant VM with

```
$ sudo vagrant up
```

## Why Use Vagrant Instead of Docker on the Host? ##

I couldn't get Docker containers on macOS to mount any directory on the file system except the user's folder.[^docker-mount].

[^docker-mount]: It's weird, because it *seems* like it should be able to do so. There are preferences for which directories can be mounted, but I never could get them to work.

Since `/etc/letsencrypt` can't be mounted in a Docker container, the certificates had to be copied either into the container or into a data volume that was mounted in the container, or into my home directory (which putting cert private keys in my home directory seems a bit sketchy).  When certs expired, as they do every 90 days with [Let's Encrypt](https://certbot.eff.org/docs/using.html#getting-certificates-and-choosing-plugins), the process had to be repeated.  Hardly difficult, but a chore to remember.[^cron]

[^cron]: I could have used cron jobs to handle all of the filesystem syncs, but I wanted something more "integrated".

Git repos mounted in my home directory aren't much of a security concern, but the Docker `osxfs` file system is a performance concern, so it was easier to put them in a Vagrant VM that was mounted to the host by NFS.

These issues present, I decided to use a more "native" approach, which is to run Docker on Linux, in a Linux VM in this case. (I know, it's funny to say running in a VM is more "native".)  While Docker can't mount the `/etc/letsencrypt` directory in macOS, it can do so on Linux. The Vagrant VM in turn can access any folder on the Mac via NFS.

# To Do #

* Windows host support using SMB shares instead of NFS
* Automatic certificate renewal?
  * This may be better served as a separate entity since they are shared with the host
* Replace nginx with HAProxy as the reverse proxy
* Try [Gitea](http://gitea.io) instead of Gogs since it sees more active development
* Running vagrant without using sudo
