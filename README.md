# Kaldi cu-device.cc memory leak ?

We've come across a memory leak when using nnet-train-simple, we have tracked it 
down to cu-device.cc not calling `cudaDevieReset()` on exit.

The affects of this bug are quite severe, during our training stage we see
around 30MB of memory lost for each nnet iteration.

Over the number of iterations in our training scripts we consume all memory on
the machine and cannot complete training.

The very strange thing is that this memory is never reclaimed at program exit
and requires a machine reboot to retrieve.

Below is extra info, some steps to reproduce and a fix that we have tested
implemented and managed to complete our training with.

It seems that this bug was fixed after updating the NVIDIA driver to version 364
(http://superuser.com/questions/1062929/freeing-kernel-memory-leaked-by-nvidia-driver).
But the latest driver version for Power8 is 352.93 and the bug is still there.

## Steps to reproduce ...

### Git LFS

This project uses git lfs (https://git-lfs.github.com/) ...

    git lfs pull

### Git submodules

This project has submodules ...

    git submodule init
    git submodule update

## Compile Kaldi versions

    It's necessary to change some files and add another ones. Please, follow the these instructions: 

    cd kaldi-upstream/tools
    awk 'CXX = g++ { print; print "CXXFLAGS += -fPIC -O3 -mcpu=power8 -mtune=power8"; next }1' Makefile
    sed -i '/# $(MAKE) PREFIX=`pwd`\/OpenBLAS\/install FC=gfortran $(fortran_opt) DEBUG=1 USE_THREAD=1 NUM_THREADS=64 -C OpenBLAS all install/c\$(MAKE) PREFIX=`pwd`\/OpenBLAS\/install FC=gfortran $(fortran_opt) DEBUG=1 USE_THREAD=1 NUM_THREADS=64 -C OpenBLAS all install' Makefile
    sed -i '/$(MAKE) PREFIX=`pwd`\/OpenBLAS\/install FC=gfortran $(fortran_opt) DEBUG=1 USE_THREAD=0 -C OpenBLAS all install/c\# $(MAKE) PREFIX=`pwd`\/OpenBLAS\/install FC=gfortran $(fortran_opt) DEBUG=1 USE_THREAD=0 -C OpenBLAS all install' Makefile
    mv ../../extras/config.sub .
    mv ../../config.guess .
    make -j 8
    extras/install_openblas.sh
    cd ../src
    mv ../../extras/configure .
    mv ../../extras/linux_ppc64le_openblas.mk makefiles/    
    ./configure --openblas-root=../tools/OpenBLAS/install
    make depend -j 8
    make -j 8

### Environment

    $ uname -a

    Linux 4.2.0-38-generic #45~14.04.1-Ubuntu SMP Thu Jun 9 09:28:10 UTC 2016 ppc64le ppc64le ppc64le GNU/Linux
    
    $ cat /proc/driver/nvidia/version

    NVRM version: NVIDIA UNIX ppc64le Kernel Module  352.93  Tue Apr  5 17:31:42 PDT 2016
    GCC version:  gcc version 4.8.4 (Ubuntu 4.8.4-2ubuntu1~14.04.3)

    $ nvidia-smi --query-gpu=index,name --format=csv

    index, name
    0, Tesla K40m

   $ /usr/local/cuda/bin/nvcc --version
    
    nvcc: NVIDIA (R) Cuda compiler driver
    Copyright (c) 2005-2015 NVIDIA Corporation
    Built on Tue_Oct_27_14:06:16_CDT_2015
    Cuda compilation tools, release 7.5, V7.5.21

### Test case
    
    $ ./memcheck.sh | tee output_memcheck

                 total       used       free     shared    buffers     cached
    Mem:      33460224    4783424   28676800       1408        128      31424
    -/+ buffers/cache:    4751872   28708352
    Swap:      1998784     169408    1829376
    1466518866
    1466519051
    1466519233
    1466519366
    1466519499
    1466519621
    1466519851
    1466520012
    1466520185
    1466520412
    1466520564
    1466520770
    1466520875
    1466521069
    1466521257
    1466521425
    1466521567
    1466521716
    1466521867
    1466522063
    1466522247
    1466522472
    1466522663
    1466522831
    1466523024
    1466523237
    1466523399
    1466523581
    1466523808
    1466523996
    1466524262
                 total       used       free     shared    buffers     cached
    Mem:      33460224    7129024   26331200       1408        128      30272
    -/+ buffers/cache:    7098624   26361600
    Swap:      1998784     169408    1829376
