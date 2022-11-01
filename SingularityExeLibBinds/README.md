# SingularityExeLibBinds

The code built in to Singularity to facilitate NVIDIA and AMD GPU usage inside containers scans standard system executable and library paths for the installed userspace libraries associated with the GPU type involved.  It then bind-mounts these files into the container file system.

On our clusters we provide multiple versions of the CUDA and ROCM suites:  this often prevents Singularity from locating the executables and libraries and the container ends up without userspace support for the GPU hardware.

Though Singularity's implicit functionality is only willing to seek-out libraries using the `ldconfig` cache for the native system, nothing prevents the user from explicitly specifying paths discovered via alternative mechanisms.  To that end, the `SingularityExeLibBinds` program will search the directories cited in the `PATH`, `LIBRARY_PATH`, and `LD_LIBRARY_PATH` environment variables; `ldconfig` cache, either system-provided or at an alternative path; and user-provided directories.

## Usage: Bind points

In its simplest form:

```(bash)
$ SingularityExeLibBinds --exe rocminfo --exe rocm-smi --lib=-lrocblas
/opt/shared/amd-roc/rocm/5.1.0/bin/rocminfo:/usr/bin/rocminfo:ro,/opt/shared/amd-roc/rocm/5.1.0/bin/rocm_smi.py:/usr/bin/rocm-smi:ro,/opt/shared/amd-roc/rocm/5.1.0/rocblas/lib/librocblas.so.0.1.50100:/usr/lib64/librocblas.so:ro
```

Since `SingularityExeLibBinds` consults the `ldconfig` cache first when resolving library names it could block paths provided on the command line or in `LIBRARY_PATH`, for example:

```(bash)
$ SingularityExeLibBinds --lib=-lboost_timer
/usr/lib64/libboost_timer.so.1.53.0:/usr/lib64/libboost_timer.so:ro

$ vpkg_require boost/1.75.0
Adding dependency `gcc/10.1.0` to your environment
Adding dependency `icu4c/68.1` to your environment
Adding package `boost/1.75.0` to your environment

$ SingularityExeLibBinds --lib=-lboost_timer
/usr/lib64/libboost_timer.so.1.53.0:/usr/lib64/libboost_timer.so:ro

$ SingularityExeLibBinds --lib=-lboost_timer --no-ldconfig
/opt/shared/boost/1.75.0/src/stage/lib/libboost_timer.so.1.75.0:/usr/lib64/libboost_timer.so:ro
```

It's also possible to ignore the `ldconfig` cache and environment variables and only use paths provided on the command line:

```(bash)
$ SingularityExeLibBinds --no-ldconfig --no-env-library-path \
    --library-path /opt/shared/boost/1.75.0/lib \
    --lib libboost_thread.so
/opt/shared/boost/1.75.0/src/stage/lib/libboost_thread.so.1.75.0:/usr/lib64/libboost_thread.so:ro
```

### Names from files

The `SingularityExeLibBinds` program can also accept lists of names of executables or libraries read from files.  A name file can contain hash-delimited comments and should have a single name per line:

```(bash)
$ cat examples/exe.txt 
#
# Each line should contain a single exectuable name:
#
ls
cat  # Lines can have terminal comments, as well
cp
mv

$ SingularityExeLibBinds --exe-from-file=examples/exe.txt 
/usr/bin/mv:/usr/bin/mv:ro,/usr/bin/cp:/usr/bin/cp:ro,/usr/bin/ls:/usr/bin/ls:ro,/usr/bin/cat:/usr/bin/cat:ro
```

## Usage: Overlays

As the number of executables and libraries grows, bind points become a less efficient means to make files available inside a container.  Singularity allows the user to provide zero or more *file system overlays* that add files atop the base container image in the runtime.  Typically this feature is used to mount a single writable overlay which captures all runtime alterations to the base image.  But any number of read-only overlays can also be layered atop the base image.

The `SingularityExeLibBinds` program by default operates in **bind-points** mode, but it can also operate in **overlay** mode.  Overlay mode builds a virtual root file system with the desired executables and libraries copied into it (e.g. at /usr/bin and /usr/lib64); an EXT3 image is then created which contains that file system.  The resulting EXT3 image can be overlayed on a Singularity container and the executables and libraries will appear to reside in the container OS (e.g. at /usr/bin and /usr/lib64).

```(bash)
$ ./SingularityExeLibBinds -vvv --mode=overlay --overlay-path=example.img \
    --exe-from-file examples/exe.txt \
    --lib='/opt/shared/netcdf/4.7.4/lib/lib*.so*'
DEBUG   : Building executable names to search
DEBUG   :   Executable names to search: mv, cp, ls, cat
DEBUG   : Building library names to search
DEBUG   :   Library names to search: /opt/shared/netcdf/4.7.4/lib/libnetcdf.so.18, /opt/shared/netcdf/4.7.4/lib/libh5bzip2.so, /opt/shared/netcdf/4.7.4/lib/libnetcdf.so, /opt/shared/netcdf/4.7.4/lib/libnetcdf.so.18.0.0
DEBUG   : Building executable search path
DEBUG   :   - Including value of $PATH
WARNING :     No such directory `/home/1001/.local/bin'
DEBUG   :   - Including default system paths
DEBUG   : Searching for 4 executables
DEBUG   :   Trying /home/1001/bin/mv
DEBUG   :   Trying /usr/local/bin/mv
DEBUG   :   Trying /usr/bin/mv
DEBUG   :     Found /usr/bin/mv (dst /usr/bin/mv)
DEBUG   :   Trying /home/1001/bin/cp
DEBUG   :   Trying /usr/local/bin/cp
DEBUG   :   Trying /usr/bin/cp
DEBUG   :     Found /usr/bin/cp (dst /usr/bin/cp)
DEBUG   :   Trying /home/1001/bin/ls
DEBUG   :   Trying /usr/local/bin/ls
DEBUG   :   Trying /usr/bin/ls
DEBUG   :     Found /usr/bin/ls (dst /usr/bin/ls)
DEBUG   :   Trying /home/1001/bin/cat
DEBUG   :   Trying /usr/local/bin/cat
DEBUG   :   Trying /usr/bin/cat
DEBUG   :     Found /usr/bin/cat (dst /usr/bin/cat)
DEBUG   : Building library search path
DEBUG   :   Issuing command /usr/sbin/ldconfig -p
DEBUG   :   - Found 2292 mapping from ldconfig cache
DEBUG   :   - Including value of $LIBRARY_PATH
DEBUG   :   - Including value of $LD_LIBRARY_PATH
DEBUG   :   - Including default system paths
DEBUG   : Searching for 4 libraries
DEBUG   :   Trying path /opt/shared/netcdf/4.7.4/lib/libnetcdf.so.18
DEBUG   :     Found /opt/shared/netcdf/4.7.4/lib/libnetcdf.so.18.0.0 (dst /usr/lib64/libnetcdf.so.18.0.0)
DEBUG   :     Symlink /opt/shared/netcdf/4.7.4/lib/libnetcdf.so.18 at /usr/lib64/libnetcdf.so.18
DEBUG   :   Trying path /opt/shared/netcdf/4.7.4/lib/libh5bzip2.so
DEBUG   :     Found /opt/shared/netcdf/4.7.4/lib/libh5bzip2.so (dst /usr/lib64/libh5bzip2.so)
DEBUG   :   Trying path /opt/shared/netcdf/4.7.4/lib/libnetcdf.so
DEBUG   :     Symlink /opt/shared/netcdf/4.7.4/lib/libnetcdf.so at /usr/lib64/libnetcdf.so
DEBUG   :   Trying path /opt/shared/netcdf/4.7.4/lib/libnetcdf.so.18.0.0
INFO    : Building overlay EXT3 image in example.img
INFO    :   Initialized with 4096 x 4k blocks, 128 inodes
INFO    : Resizing overlay EXT3 image in example.img
INFO    :   Compacted to 1750 (4k) blocks
Use the overlay by adding a `--overlay example.img:ro' flag to your singularity commands
```

The overlay can then be used in a Singularity container thusly:

```(bash)
$ singularity exec --overlay example.img:ro /opt/shared/singularity/prebuilt/postgresql/13.2.simg /bin/bash

Singularity> find /usr/lib64 -name '*netcdf*'
/usr/lib64/libnetcdf.so.18.0.0
/usr/lib64/libnetcdf.so.18
/usr/lib64/libnetcdf.so
```
