
casync application container
----------------------------

*This is not an "official" project and not part of casync or systemd*

It solved my problem, if it's useful for someone else that's great.

----

**Why?**

I kept wanting to use [casync](https://github.com/systemd/casync)
on old systems that do not provide a package for it.

I then found that compiling casync from source is a challenge because it
has several modern dependencies.  Even after tracking down libs and build
tools, issues can boil down to a libc version being too old on the old
distro, and I'm not up for messing with those details on a system in
production.

So I use this application container as a stopgap solution, since I **do**
usually have Docker on those older machines.


The **Dockerfile** is trivial of course (see also
[DesignNotes.txt](https://github.com/gunnarx/casync-docker/blob/master/DesignNotes.txt))
and the useful stuff is mostly the content of the **Makefile** and the
defaults given there for the "make run" command.

Author
------
Copyright (C) 2018 Gunnar Andersson

License
-------
Your choice of MPLv2 (license text [provided](https://github.com/gandaman/casync-docker/blob/master/LICENSE)), GPLv2 or GPLv3.

Contributing
------------

- Send a Pull Request here on GitHub for master branch.
- Agree that you are contributing under the project's original license
- Sign your commit

_I get a lot of notifications and they get filtered. So if I miss it and
don't answer quickly - just send me a direct email._

Dependencies
------------
- Docker and (GNU) make.

Building Docker Image
---------------------

- View the Makefile to see the environment variables.
- Edit the variables to your liking or override temporarily by exporting new values in your environment.

Then:

```
make build
```

Running casync
--------------

The Makefile does the heavy lifting (Do you find this odd and prefer a
script?  I don't disagree... See TODOs for discussion and send me a pull
request). 

Environment variables are used to get arguments into the Makefile.

There are three separate bind mounts possible from host to container:
They can be used optionally but you will need to use at least one to
process data.  They all default to the directory you are currently in
($PWD) if not specified.

- WORK_DIR
- OUTPUT_DIR
- INPUT_DIR

They are mounted on /workdir, /outputdir and /inputdir respectively.

The casync command is placed into
- CASYNC_ARGS

Examples
--------

Run casync --help
(this works because the shell passes the variable assignment in environment)

```bash
CASYNC_ARGS=--help make run
```

or if this order makes more sense to you:
(this works because Makefile accepts variable definitions in the input parameters)
```
make run CASYNC_ARGS="--help"
```

At this point, you start feeding in some data.  If you are making use of
the fact that the working directories default to your current directory
then you are probably not standing in the casync-docker directory to do
useful work.  Remember to then point out the Makefile with a -f parameter,
as such:

```
make run -f <path-to-casync-docker>/Makefile
```

OK, let's look at some more. 
*NOTE* On the next two examples don't get confused by the fact that
"**make**" is one of the verbs (commands) given to the casync binary,  and
that we also write "make" to call the Makefile.  The second make is just the
command actually passed to casync.

Use "casync make" to create a **catar archive** of /usr of the container
itself (not that useful, but it's a simple example):

```
make run CASYNC_ARGS="make /workdir/usr.catar /usr"
: Remember /workdir defaults to your current directory on the host
: The created file is now here, on your host:
ls -l ./usr.catar
```

Use "casync make" to create a **casync index** of the same and store the
casync store in ./mystore/:
```
make run CASYNC_ARGS="make /workdir/usr.caidx --store=/workdir/mystore /usr"
```

Create a casync index of your the Photos folder in /mnt/networkdisk and put
the corresponding data store in $HOME/backupstore/. In this example, we use
export to set environment first instead of a temporary assignment.

```
export INPUT_DIR=/mnt/networkdisk
export WORK_DIR=/home/myself/
export OUTPUT_DIR=/home/myself/backupstore
export CASYNC_ARGS="make /workdir/Photos.caidx --store=/outputdir/Photos.castr /inputdir/Photos/"
make run
```

Restore the file tree from the previously created casync index of your
Photos folder into $HOME/temp_pictures.  This time we are more lazy and
just use the /workdir for all input and output, and we rely on that work
dir defaults to our current working directory:

```
unset INPUT_DIR WORK_DIR OUTPUT_DIR # (Reset your env, just in case you used export above)
cd $HOME
mkdir temp_pictures  # Target dir should exist
: Note $WORK_DIR is empty and will therefore default to current directory on host ($HOME)
export CASYNC_ARGS="extract --store=/workdir/backupstore/Photos.castr /workdir/backupstore/Photos.caidx /workdir/temp_pictures"
make run -f casync-docker/Makefile
```

Obviously the store can be an online store, and there are many other options.

OK, that should be enough examples because you either know how to use
casync and have learned this on a much less complicated setup, or you need
to go and learn that first.  Then it should be easy to apply that to this
setup.

Please refer to [casync](https://github.com/systemd/casync) resources for
more information.

Troubleshooting
---------------

*Failed to run synchronizer: Permission denied*

1. You might have forgotten to create one of the mount directories, such as
OUTPUT_DIR? If the directory does not exist it seems to be created by
docker before mounting and it then ends up being owned by root, so casync
cannot write to it. (Note that casync is run with a user ID equal to your
own when you launch, so that the written files are also owned by your
user. Permissions must be set accordingly)

2. Other weird permission issues are usually SELinux related, if you are
running a distro that uses SELinux.  The container itself might not be
allowed to write (or even read) to the mounted host directory. Please
google "docker selinux permission denied" or similar for some references.

For casync in general you might run into that SElinux attributes have been
stored in the casync archive, and those might not be supported in the
extract environment, or similar issues. (Read the manual - casync is super
strict about reproducibility).  If you don't care about those and just want
the files you could try passing --without=selinux to casync on either the
archiving or extracting step.  (But seriously, go back to casync docs
and online resources - there are more things like ACLs and extended
attributes in general to consider).


Bugs/unsupported
----------------

- **casync mount** has not been implemented -- see TODOs
- Other networking (such as an online store) not really tested  - please try and report back
- Haven't tested block devices - please try and report back
- There is no man installed.  Of course the container is not made to be interactive, but it could be useful to have it on the image and support some way to view it. Here is an [online man page](https://man.cx/casync) for you.

Security disclaimer
-------------------

- The container is intended to run with the personal user id running the
binaries and not root and it bind-mounts only the directories as documented
above.  It does not run in Docker's --privileged mode.  That said, I have
NOT analyzed if there is some way that the binaries run with higher
privilege than expected.  Considering what Docker can do, it's possible
that there are some corner cases and YOU RUN THIS AT YOUR OWN RISK.

TODOs/Wishlist
--------------

1. Shrink container, use for example Alpine Linux -- see DesignNotes.txt

2. Create a shell script that wraps the somewhat ugly "make run" command.
User experience should mimic a standard casync installation, to the
extent possible:

```
./casync <normal casync arguments...>
```

_Discussion_

Now, I like having everything about a docker container in single a Makefile
as a starting point, but a script would help usability.  The reason I
just accept the current way for now is that I rarely run casync manually in
the shell anyway, so it will probably be called from some other shell
script and then I can just deal with the bad user experience at the time
when it is written.  This just needs to work and give access to casync.

Note however, if that wrapper is to mimic the standard casync binary and
the arguments refer to local (host) files, then it would either have to
smartly figure out which host dirs to bindmount and fiddle with the
arguments to the container, or it might have to provide the entire host VFS
to the container.  (See Security Disclaimer section for some concern I
have).

3. Support **casync mount**.  Unlike the current container that runs and
exits , and is then removed again, this oe might need to detach and keep
running, and make the mount available to the host.  I have not looked at
it...there might be some obstacles to overcome. -- Maybe you will try it,
and send me a pull request?

4. Others?
Any idea you might have.  For example I'd happily support some other
runtime (and rename the project accordingly):

- **rkt**
- **runc**
- **systemd-nspawn** (I guess it's a small subset of systems that support systemd-nspawn but not casync, but who knows)
- **flatpak** ? (flatpak developers typically respond with that flatpak is not intended for CLI tools)

References/Other
----------------

- Another way to go might be [desync](https://github.com/folbricht/desync).
- Having a fully statically linked casync would be awesome - please let me
know if you know of one, or how to create one.


