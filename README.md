# SingularityTools

Helper utilities for making use of Singularity containers.

## SingularityExeLibBinds

The code built in to Singularity to facilitate NVIDIA and AMD GPU usage inside containers scans standard system executable and library paths for the installed userspace libraries associated with the GPU type involved.  It then bind-mounts these files into the container file system.

On our clusters we provide multiple versions of the CUDA and ROCM suites:  this often prevents Singularity from locating the executables and libraries and the container ends up without userspace support for the GPU hardware.

Though Singularity's implicit functionality is only willing to seek-out libraries using the `ldconfig` cache for the native system, nothing prevents the user from explicitly specifying paths discovered via alternative mechanisms.  To that end, the `SingularityExeLibBinds` program will search the directories cited in the `PATH`, `LIBRARY_PATH`, and `LD_LIBRARY_PATH` environment variables; `ldconfig` cache, either system-provided or at an alternative path; and user-provided directories.

### Usage

In its simplest form:

```(bash)
$ SingularityExeLibBinds --exe rocminfo --exe rocm-smi --lib=-lrocblas
/opt/shared/amd-roc/rocm/5.1.0/bin/rocminfo:/usr/bin/rocminfo:ro,/opt/shared/amd-roc/rocm/5.1.0/bin/rocm_smi.py:/usr/bin/rocm-smi:ro,/opt/shared/amd-roc/rocm/5.1.0/rocblas/lib/librocblas.so.0.1.50100:/usr/lib64/librocblas.so:ro
```

Since `SingularityExeLibBinds` consults the `ldconfig` cache first when resolving library names it could block paths provided on the command line or in `LIBRARY_PATH`, for example:

```(bash)
$ ./SingularityExeLibBinds --lib=-lboost_timer
/usr/lib64/libboost_timer.so.1.53.0:/usr/lib64/libboost_timer.so:ro

$ vpkg_require boost/1.75.0
Adding dependency `gcc/10.1.0` to your environment
Adding dependency `icu4c/68.1` to your environment
Adding package `boost/1.75.0` to your environment

$ ./SingularityExeLibBinds --lib=-lboost_timer
/usr/lib64/libboost_timer.so.1.53.0:/usr/lib64/libboost_timer.so:ro

$ ./SingularityExeLibBinds --lib=-lboost_timer --no-ldconfig
/opt/shared/boost/1.75.0/src/stage/lib/libboost_timer.so.1.75.0:/usr/lib64/libboost_timer.so:ro
```

It's also possible to ignore the `ldconfig` cache and environment variables and only use paths provided on the command line:

```(bash)
$ ./SingularityExeLibBinds --no-ldconfig --no-env-library-path \
    --library-path /opt/shared/boost/1.75.0/lib \
    --lib libboost_thread.so
/opt/shared/boost/1.75.0/src/stage/lib/libboost_thread.so.1.75.0:/usr/lib64/libboost_thread.so:ro
```

#### Names from files

