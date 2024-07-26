# [Yocto]  Builds without an Internet Connection

# Introduction

Yocto requires internet access in order to download all the source packages required to build an embedded Linux system.  However, internet access may not always be available when building with Yocto, expecially using network in China.  A developer may have a build machine with internet restrictions or the developer may simply be unplugged while on an airplane.  This wiki covers a few options for building with offline machines.

# Downloads and Shared State Cache Refresher

Two important directories in the `build` directory are the `downloads` and `sstate-cache`.

The `downloads` directory is where all the source packages are downloaded locally.  This directory may be relocated with the `DL_DIR` variable in your `local.conf`.

```bash
DL_DIR ?= "${TOPDIR}/downloads"
```

The `sstate-cache` directory is where all the build artifacts are cached which can be reused to accelerate subsequent builds.  This directory may be relocated with the `SSTATE_DIR` variable in your `local.conf`.

```bash
SSTATE_DIR ?= "${TOPDIR}/sstate-cache"
```

For example, if you want to place the `downloads` and `sstate-cache` directories outside of the standard Yocto build directory, you may edit your `local.conf` as shown below.

```bash
DL_DIR = "/home/<user>/yocto/downloads"
SSTATE_DIR = "/home/<user>/yocto/sstate-cache"
```

# Downloading Source Packages

At some point, you will need a Yocto machine that has internet access to download all the source required for a build.  You may download all the source packages (without building) by issuing `bitbake` with the `fetchall` command as shown in the `fetchall` listing below.  All the source packages required for your build are now located in the `downloads` directory.  You may generate tarballs from the Git repositories by setting the `BB_GENERATE_MIRROR_TARBALLS` flag to "1" in your `local.conf` if desired.

`$ bitbake -c fetchall core-image-minimal`

If you want to download every package, you can `fetchall` the `world`.

`$ bitbake -c fetchall world

# No Network Access

If you are working on a machine with no network access, you may tar the downloads directory (including .done files) from an internet connected machine or mirror server and copy the tarball to your build machine.  Then extract the source packages into the downloads directory.  Internet accesses can be disabled on the build machine by setting the `BB_NO_NETWORK` flag in local.conf.  The Yocto build will immediately throw an error if it tries to access the network.

`BB_NO_NETWORK = "1"`

# No Internet Access

## Mirror Server

If you are connected to a local network, but do not have internet access, your IT department may setup a mirror server to provide the source packages.  Fetching source packages from a local mirror server, requires network access, but only the mirror server requires internet access.  Set the `BB_GENERATE_MIRROR_TARBALLS` flag in local.conf to generate tarballs from Git repositories on the mirror server.  This is only required on the mirror server and not on the developer's machine.

`BB_GENERATE_MIRROR_TARBALLS = "1"`

If you are administering a mirror server, you may download all the source packages (without building) by issuing bitbake with the fetchall command as shown in the fetchall listing above.  These packages will be available to all developers with LAN access to the mirror server.

On the developer's machine, point to the mirror server with the `SOURCE_MIRROR_URL` variable and force fetching source from the mirror server by setting the `BB_FETCH_PREMIRRORONLY` variable in local.conf. Any requests that cannot be serviced by the mirror server will throw an error.

local.conf (developer's machine):
```
MIRROR_SERVER = "http://your-mirror-server/"
 
BB_FETCH_PREMIRRORONLY = "1"
 
INHERIT += "own-mirrors"
SOURCE_MIRROR_URL = "${MIRROR_SERVER}/downloads/"
UNINATIVE_URL = "${MIRROR_SERVER}/downloads/"
```

## Build Server

A build server is a machine that has internet access and performs periodic Yocto builds.  These builds may be scheduled at intervals such as nightly or weekly to keep the developers coherent.  Developers can fetch the build artifacts from the shared state cache of the build server significantly reducing local build times.

The configuration below will first fetch from the shared state cache if there is a hit and then attempt to download source from the mirror server if the package is not cached or stale.

```
MIRROR_SERVER = "http://your-mirror-server/"
 
BB_FETCH_PREMIRRORONLY = "1"
SSTATE_MIRROR_ALLOW_NETWORK = "1"
 
INHERIT += "own-mirrors"
SOURCE_MIRROR_URL = "${MIRROR_SERVER}/downloads/"
UNINATIVE_URL = "${MIRROR_SERVER}/downloads/"
 
SSTATE_MIRRORS ?= " \
    file://.* ${MIRROR_SERVER}/sstate-cache/PATH \
"
```

# Conclusion

While Yocto requires internet access at some point to fetch all the required source packages to build an embedded distribution, it does provide tools that allow building on machines that do not have internet access.  A couple of options for building without internet access include:

Downloading source packages and then copying them to your build machine.
Using a mirror server to host the source packages locally on your intranet.
Using a build server to run periodic builds and provide the build artifacts through the shared state cache.
