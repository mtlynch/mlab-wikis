

## OVERVIEW ##

If you are an M-Lab experiment developer, this document will help you
prepare your experiment for deployment to M-lab.

[https://code.google.com/p/m-lab/source/browse/slicebasemanager/slice_example?repo=ops
This directory] is a working example of an experiment used on M-Lab.

## PROVIDED SERVICES ##

The following services are guaranteed to be running prior to installation of
your experiment package:

  * crond    -- logs all script output to `/var/log/cron.output`, not mail.
  * rsyslogd -- allows crond and other services to log to `/var/log/cron`, et al
  * rsync --daemon -- reads from `/var/spool/$SLICENAME/` on port 7999 for slices with a dedicated IP address.

## PROVIDED FUNCTIONS & COMMANDS ##

The following scripts and commands are guaranteed to be installed prior to installation of
your experiment package:

```
/etc/slice-functions
```
> The script `/etc/slice-functions` defines convenience functions in bash for configuring your experiment.
    * `get_slice_ipv4()` -- the IPv4 address assigned to this slice.
    * `get_slice_ipv6()` -- the IPv6 address assigned to this slice (if any)
    * `get_slice_name()` -- the name of this slice.  See also $SLICENAME.
    * `get_site_name()`  -- The site name, i.e. LGA02, NUQ01, etc.
    * `$SLICENAME`       -- predefined to value of `get_slice_name()`
    * `$SLICEHOME`       -- same as user's `$HOME`, `$SLICEHOME` is used in these docs because this is also the content of the experiment package unpacked into the $HOME dir.

```
/usr/bin/slice-restart
```
> The command `/usr/bin/slice-restart` calls out to the root environment to fully shutdown your slice, then start it back up again.  This effectively simulates a machine restart.

```
/usr/bin/slice-recreate
```
> The command `/usr/bin/slice-recreate` calls out to the root environment to delete and then create a new version of your slice.  This effectively simulates a reinstall.

```
/etc/init.d/slicectrl
```
> The slicectrl init-script is the wrapper that will ultimately invoke the experiment-provided commands for starting, stopping the experiment through the init-script mechanisms.

## EXPERIMENT PACKAGE REQUIREMENTS ##

The format and conventions expected from the experiment package reduce to two basic things:

  1. The root directory of the package should contain a directory named `init`.
  1. The `init` directory should contain action scripts for your experiment, detailed below.

Beyond these two requirement, your experiment package may include any
additional files and directories that you desire.

### Package Creation ###

The experiment package is expected to have an `init` directory at the root of
the archive.  The simplest command to create a package in this format is:

```
    tar -C <slicedir> -zcvf <slicename>.tar.gz init [ <dir1>, <fileN>, ... ]
```

Where `<slicedir>` is the working directory of your experiment build, and `<slicename>` is the PlanetLab name of your slice, i.e. mlab\_ops, iupui\_ndt, or mpisws\_broadband.

### Package Upload to M-Lab ###

We will provide a script to sign the slice package using the developer's private SSH key.  After uploading the package and signature to M-Lab, the signature and package would be verified using the developer's public SSH keys.

These steps preserve an explicit and verifiable chain of custody from developer to M-Lab nodes.

TODO: provide a mechanism for uploading experiment package.

  * web form drop box: receives package & signature, verifies package, and places in slice-packages dir.
  * a more interactive web interface could be build around this drop box.

### Package Installation ###

The experiment package is unpacked into $SLICEHOME, and then every file is checksummed.  The checksum is saved and all files are marked read-only.

Therefore, unpackaged experiment files should remain UNCHANGED.  If any
modifications are needed to files contained in the experiment package, they
should be copied outside of the experiment package and installed into the
filesystem.  i.e. /usr/local/  /etc/

Based on the package creation example above, after unpacking the directory layout would be:

```
    $SLICEHOME/init/
    $SLICEHOME/dir1/
    $SLICEHOME/fileN
```

And, all files would be read-only.

### Package Scripts ###

Your experiment package should define the scripts described below.  These are
the entry points used by `service slicectrl` to manage your experiment.

The scripts in `$SLICEHOME/init/` will always run as the SLICE USER, not as
root.  Therefore, every action that requires elevated privileges must be taken
using `sudo` or a similar mechanism.

When a script executes, the PWD will be $SLICEHOME. This is the same as the
home directory of the slice user and root of the slice package contents.

Scripts in experiment package:

```
$SLICEHOME/init/initialize
```
> Presence: optional.

> If your experiment requires additional one-time configuration such as
> installation of dependent packages, setup of config files in httpd or
> other, this is the place to do it.

> The initialize script should be idempotent; it should be possible to run
> it repeatedly without ill effect.  `initialize` will be run every time a
> new version of your slice package is installed.  But, if package
> dependencies have already been installed, then it is not necessary to
> repeat them, or new dependencies may have been added in the new version.

> If initialize fails, it should return a non-zero exit status.  This will
> prevent the experiment from continuing with an un-initialized environment.

```
$SLICEHOME/init/start
```
> Presence: required

> This script should start your service and any processes needed by your
> service.  Running `start` more than once should be well behaved.  i.e.
> check that process is not running before executing:
> `pgrep $SERVICE_NAME > /dev/null || ($SERVICE_NAME &)`

```
$SLICEHOME/init/stop
```
> Presence: required

> This script should gracefully stop any processes that were started by your
> service.

```
$SLICEHOME/init/status
```
> Presence: optional, encouraged.

> You are encouraged to provide a meaningful indication of service health
> and liveness.  This will assist you when observing the status of your
> service across M-Lab and it will assist M-Lab operators who may see and
> respond to issues first.

> A minimal example script is included
> [here](https://code.google.com/p/m-lab/source/browse/experimentbase/slice_example/init/status?repo=ops)

> The status script will be called periodically, i.e. once an hour.

> The `status` script should print a one-line status message and have exit
> status of 0, 1, 2, 3 corresponding to OK, WARNING, CRITICAL, or UNKNOWN.
> As well the script should return in less than 60 seconds.

> A minimal example might verify the presence of a server process.  A better
> example might check for liveness of open ports by a server process.  An
> excellent example might run a live test of the service using the client
> protocol.