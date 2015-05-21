# Introduction #

This describes the process of building a package (RPM) for Paris Traceroute.

# Details #

In summary, process is to follow the instructions at https://code.google.com/p/paris-traceroute/wiki/Packages, for Debian (even though building an RPM).

  * git clone -b jc https://code.google.com/p/paris-traceroute.libparistraceroute
  * cd paris-traceroute.libparistraceroute/libparistraceroute
  * ./autogen.sh
  * ./configure
  * edit spec files (find . -name `\*.spec`, then edit the results to change version numbers from 0.1 to 1.0).
  * make rpm

# Known problems #

**UDP mode doesn't appear to work from slide (no response to probes). ICMP mode does work.**

# RPM Repository #

Add the following content to /etc/yum.repos.d in a file with a **.repo extension.
```
[MeasurementLabCentos]
name=MLab
baseurl=http://boot-test.measurementlab.net/install-rpms/MeasurementLabCentos
gpgcheck=1
gpgkey=http://boot-test.measurementlab.net/PlanetLabConf/get_gpg_key.php
```**

Then you can install paris-traceroute from the repository:
```
    yum clean all; yum clean metadata;
    yum install paris-traceroute
```