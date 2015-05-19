

## OVERVIEW ##

The SliceBaseManager is a component provided by M-Lab to run inside a slice
environment to ease management of an experiment.

If you are an M-Lab operator, this document will help you understand how the
SliceBaseManager fits into general M-Lab operations and slice management in
particular.

[This directory](https://github.com/m-lab/mlab-ops/tree/master/slicebasemanager/slice_example) is a working example of an experiment used on M-Lab.

## Manager Structure ##

At a high-level, SliceBaseManager is divided into three parts:

  1. initscript -- fits into the PlanetLab initscript mechanism
  1. smp -- the "slice manager package" provides dependencies common to all experiments.
  1. slice\_example -- a template and working example of an experiment package that works within the SliceBaseManager

Each of these parts is also a directory that contains the implementation.

### initscript ###

> This directory contains a default initscript for slices running on
> M-Lab.  The initscript is part of the PlanetLab initscript mechanism.
> It will be installed as /etc/init.d/vinit.slice

> It defines actions for 'start', 'stop', 'reset', and 'update'. Start and Stop
> actions are called by the NodeManager on the M-Lab machine; these correspond
> to the slice starting and stopping respectively.  Update is to update the SMP
> package, if needed.  And, Reset is to uninstall everything and restore a
> standard slice environment.

> This script is installed in the slice upon first creation.  It's purpose is to
> complete initialization of the local environment to support crond and rsyslog.
> It then downloads, verifies and installs the `smp`, a minimal set of scripts
> for managing the installation of the actual experiment package.

> On first invocation, it asserts some basic slice environment settings, i.e.
> crond is running, rsyslogd and then attempts to download the slice manager
> package.

### smp (Slice Manager Package) ###

> This directory is the slice manager package.  The slice manager installs
> some generic support services needed by all slices.  Specifically, it
> configures rsyncd, creates necessary directories, and installs generic
> bash functions for use by experiment scripts.

> The slice manager installs `/etc/init.d/slicectrl` as a standard init-script
> that will be started as part of slice init.

> `/etc/init.d/slicectrl` supports 'start', 'stop', 'status', 'initialize',
> 'update', 'restart'

> Running 'slicectrl update' checks for an experiment-specific package,
> downloads, and installs it.  If the latest version is already installed, no
> action is taken.

> TODO: add a random delay within the hour to avoid hitting central all at once.

### slice\_example ###

> This directory provides an minimal template for an experiment that will
> work with the slice manager.

> More detailed description is here: SliceBase
