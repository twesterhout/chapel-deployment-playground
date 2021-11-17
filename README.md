This repository is for experiments deploying multi-locale Chapel programs.

This README discusses the implementation, see [the discussion on Chapel
Discourse](https://chapel.discourse.group/t/chapel-deployment/7860/12) for more
context.

### Building the containers

```shell
singularity build base.sif base.def
singularity build chapel-1.25.0.sif chapel-1.25.0.def
singularity build hello-world.sif hello-world.def
```

`base` image is very similar to [the official Chapel Docker
image](https://github.com/chapel-lang/chapel/blob/main/util/dockerfiles/Dockerfile).
We add a few dependencies:

 * `libopenmpi3` and `libopenmpi-dev`
 * `libpmix2` and `libpmix-dev` although they are installed automatically as
   OpenMPI dependencies.

`chapel-1.25.0` image builds Chapel in 3 configurations:

 * with `CHPL_COMM=none` for laptop/single-node usage;
 * with `CHPL_COMM=gasnet CHPL_COMM_SUBSTRATE=ibv CHPL_GASNET_SEGMENT=fast` for
   "proper" clusters (personally, I test it on
           [Snellius](https://www.surf.nl/en/dutch-national-supercomputer-snellius)).
 * with `CHPL_COMM=gasnet CHPL_COMM_SUBSTRATE=mpi` for our local cluster where
   nodes are connected using Ethernet.

I had to explicitly set `CC=gcc CXX=g++ CF=gfortran` because otherwise Chapel
failed to find MPI (it complained about the main compiler being `clang` and
`mpicc` using `gcc` under the hood). Furthermore, setting
`PMI_HOME=/usr/lib/x86_64-linux-gnu/pmix2` was required for Chapel to compile
GASNet with PMI support.

`hello-world` image builds `hello6-taskpar-dist.chpl` example executable from
Chapel Primers. The resulting executables are
`/project/bin/{none,mpi,ibv}/hello6-taskpar-dist`. Note that all configurations
are compiled with `CHPL_LAUNCHER=none` such that GASNet doesn't run it's
launcher scripts (which I couldn't get to work).

### Running the executable

First, we can always do

```shell
singularity exec hello-world.sif /project/bin/none/hello6-taskpar-dist
```

which will run the application one one locale (tested on my laptop, local
cluster, and Snellius).

To use multiple locales, we can use mpi:

```shell
mpirun -np 3 singularity exec hello-world.sif /project/bin/mpi/hello6-taskpar-dist -nl 3
```

Note, that since our container was built with OpenMPI, `mpirun` in the above
command *must* come from OpenMPI. The above command worked on my laptop and on
Snellius, but crashed on our local cluster since OpenMPI installation there was
too old. This can be easily fixed with Conda:

```shell
conda create -n openmpi
conda activate openmpi
conda install -c conda-forge openmpi
```

This is typically frowned upon in HPC environments since MPI in Conda is not
configured to achieve the highest performance on a specific hardware. This
is not an issue for us since we will use MPI conduit only for machines connected
via Ethernet.

Finally, on a machine with Infiniband, we can do:

```shell
salloc -N 2 --ntasks-per-node=1 --exclusive srun -n 2 singularity exec hello-world.sif /project/bin/ibv/hello6-taskpar-dist -nl 2 -v
```

Example output:
```
salloc: Pending job allocation 148015
salloc: job 148015 queued and waiting for resources
salloc: job 148015 has been allocated resources
salloc: Granted job allocation 148015
salloc: Waiting for resource configuration
salloc: Nodes tcn[121,123] are ready for job
[tcn121:3457875] PMIX ERROR: ERROR in file ../../../../../../src/mca/gds/ds12/gds_ds12_lock_pthread.c at line 168
[tcn123:3755502] PMIX ERROR: ERROR in file ../../../../../../src/mca/gds/ds12/gds_ds12_lock_pthread.c at line 168
QTHREADS: Using 128 Shepherds
QTHREADS: Using 1 Workers per Shepherd
QTHREADS: Guard Pages Enabled
QTHREADS: Using 8376320 byte stack size.
executing on node 1 of 2 node(s): tcn123
QTHREADS: Using 128 Shepherds
QTHREADS: Using 1 Workers per Shepherd
QTHREADS: Guard Pages Enabled
QTHREADS: Using 8376320 byte stack size.
executing on node 0 of 2 node(s): tcn121
Hello, world! (from locale 0 of 2 named tcn121)
Hello, world! (from locale 1 of 2 named tcn123)
salloc: Relinquishing job allocation 148015
```

No idea how to fix those PMIX errors...












