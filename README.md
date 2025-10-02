# Vela_Berkeley2 Setup and Execution Guide for Lengau Cluster

## Overview

This guide provides step-by-step instructions for compiling and running the Vela_Berkeley2 astrophysical simulation code on the Lengau cluster at CHPC (Centre for High Performance Computing).

## Prerequisites

### System Requirements
- Access to Lengau cluster (CHPC account)
- SSH access to `lengau.chpc.ac.za`
- PBS job scheduler access
- Intel Parallel Studio XE compiler suite

### Account Setup
1. Ensure you have an active CHPC account
2. Verify access to the `bigmem` queue (for large memory jobs)
3. Confirm your project allocation (e.g., ASTR1245)

## Directory Structure

```
/home/cventer/lustre/Vela_Berkeley2/
├── Vela_SSC_iof_ppSC_MPI.cpp    # Main source file
├── comm.cpp                     # Communication module
├── comm.h                       # Communication header
├── options.cpp                  # Options handling
├── options.h                    # Options header
├── physproc.c                   # Physics processes
├── globals.h                    # Global definitions
├── defines.h                    # Preprocessor definitions
├── debug.c                      # Debug utilities
├── nrutil.c                     # Numerical utilities
├── timer.h                      # Timing utilities
├── Makefile                     # Build configuration
├── VHE_CHPC.job                 # PBS job script
├── parameters.dat               # Input parameters
└── *.backup                     # Backup files
```

## Compilation Setup

### 1. Environment Configuration

```bash
# Connect to Lengau
ssh -X cventer@lengau.chpc.ac.za

# Navigate to project directory
cd /home/cventer/lustre/Vela_Berkeley2/

# Clean module environment
module purge

# Load Intel toolchain
module load chpc/parallel_studio_xe/18.0.1/2018.1.038

# Verify environment
module list
which mpiicpc
```

### 2. Makefile Configuration

The optimized Makefile includes:
- Intel MPI C++ compiler (`mpiicpc`)
- Optimized flags: `-O2 -Wall -mcmodel=medium -shared-intel -qopenmp`
- OpenMP support for potential parallelization

```makefile
CPP = mpiicpc
CFLAGS = -O2 -Wall -mcmodel=medium -shared-intel -qopenmp
LDFLAGS = -qopenmp

%.o : %.cpp
	${CPP} ${CFLAGS} -c $<

Vela_SSC_iof_ppSC_MPI : Vela_SSC_iof_ppSC_MPI.o comm.o options.o
	${CPP} ${CFLAGS} ${LDFLAGS} -o $@.x $^

clean:
	rm *.o *.x

cleanall:
	rm *.o *.x caps.dat logrings.dat ph_en_bin.dat phres.dat SRemis.dat total_spectra.dat
```

### 3. Compilation Process

```bash
# Clean previous builds
make clean

# Compile the code
make

# Verify compilation
ls -la Vela_SSC_iof_ppSC_MPI.x
file Vela_SSC_iof_ppSC_MPI.x
```

**Expected Output:**
- Executable: `Vela_SSC_iof_ppSC_MPI.x` (~326KB)
- Minor warnings about format strings (non-critical)
- No compilation errors

## Job Submission

### 1. PBS Job Script Configuration

The `VHE_CHPC.job` script is configured for:
- **Queue:** `bigmem` (high memory requirements)
- **Resources:** 56 CPUs, 980GB RAM, 48-hour walltime
- **Project:** ASTR1245
- **Email notifications:** christo.venter7@gmail.com

```bash
#!/bin/bash
#PBS -l select=1:ncpus=56:mpiprocs=56:mem=980gb
#PBS -P ASTR1245
#PBS -q bigmem
#PBS -W group_list=bigmemq
#PBS -l walltime=48:00:00
#PBS -o test.out
#PBS -e test.err
#PBS -m abe
#PBS -M christo.venter7@gmail.com

ulimit -s unlimited

# Clean module environment and load only Intel toolchain
module purge
module load chpc/parallel_studio_xe/18.0.1/2018.1.038

cd /mnt/lustre/users/cventer/Vela_Berkeley2/

nproc=`cat $PBS_NODEFILE | wc -l`
echo nproc is $nproc
cat $PBS_NODEFILE

echo " my loaded modules are"
echo `module list`

echo "WORKING DIRECTORY" VHE.job
echo `pwd`
echo " "

# Clean and rebuild
echo "CLEANING PREVIOUS BUILD"
make clean
echo " "

echo "COMPILING WITH NEW SETTINGS"
make
echo " "

if [ -f "Vela_SSC_iof_ppSC_MPI.x" ]; then
    echo "COMPILATION SUCCESSFUL - RUNNING PROGRAM"
    mpirun -np $nproc ./Vela_SSC_iof_ppSC_MPI.x < parameters.dat > stdout.log
else
    echo "COMPILATION FAILED - CHECK ERROR MESSAGES"
    exit 1
fi

exit 0
```

### 2. Submitting Jobs

```bash
# Submit the job
qsub VHE_CHPC.job

# Check job status
qstat -u cventer

# Monitor job output
tail -f test.out
tail -f test.err
```

### 3. Input Parameters

The `parameters.dat` file contains simulation parameters:
```
delring = 0.003
outrad = 0.96
inrad = 0.9
numel = 1
azim_divs = 1
pc_cover_mode = 0
code_check = 2
```

## Output Files

After successful execution, the following files will be generated:
- `stdout.log` - Program output and diagnostics
- `caps.dat` - Simulation results
- `logrings.dat` - Ring logging data
- `ph_en_bin.dat` - Photon energy binning
- `phres.dat` - Photon resolution data
- `SRemis.dat` - Synchrotron emission data
- `total_spectra.dat` - Total spectral output

## Troubleshooting

### Common Issues and Solutions

#### 1. Compilation Errors

**Problem:** Missing separator errors in Makefile
```bash
Makefile:6: *** missing separator. Stop.
```

**Solution:** Ensure Makefile uses tabs, not spaces for indentation:
```bash
# Recreate Makefile with proper tabs
rm Makefile
# Use the provided Makefile template above
```

#### 2. Module Conflicts

**Problem:** Mixed Intel/GCC modules causing conflicts
```bash
# Error: Incompatible compiler versions
```

**Solution:** Use clean module environment:
```bash
module purge
module load chpc/parallel_studio_xe/18.0.1/2018.1.038
```

#### 3. Memory Issues

**Problem:** Core dumps or segmentation faults
```bash
# Multiple core.XXXX files (8.9GB each)
```

**Solution:** 
- Use `-mcmodel=medium` instead of `-mcmodel large`
- Ensure sufficient memory allocation in PBS script
- Check for array bounds and memory leaks

#### 4. MPI Runtime Errors

**Problem:** Process manager errors
```bash
[mpiexec@fat04] control_cb (pm/pmiserv/pmiserv_cb.c:208): assert (!closed) failed
```

**Solution:**
- Use consistent MPI implementation (Intel MPI)
- Avoid mixing MPI implementations
- Check process count matches available cores

#### 5. Job Queue Issues

**Problem:** Job not starting or queued indefinitely

**Solutions:**
- Check queue availability: `qstat -q`
- Verify resource requirements match queue limits
- Ensure project allocation is sufficient
- Check for typos in PBS directives

### Performance Optimization

#### 1. Compiler Optimization
- Use `-O2` for balanced performance/stability
- Avoid `-O3` for complex simulations
- Enable OpenMP with `-qopenmp` for shared memory parallelization

#### 2. Memory Management
- Use `-mcmodel=medium` for large datasets
- Set `ulimit -s unlimited` for stack size
- Monitor memory usage with `top` or `htop`

#### 3. MPI Configuration
- Match process count to available cores
- Use Intel MPI for best performance with Intel compiler
- Consider process binding for NUMA optimization

## Best Practices

### 1. Development Workflow
1. Always test compilation before submitting jobs
2. Use small test cases for debugging
3. Keep backup copies of working configurations
4. Document any custom modifications

### 2. Resource Management
1. Request appropriate memory (980GB for this simulation)
2. Use walltime estimates based on previous runs
3. Monitor job progress and adjust resources as needed
4. Clean up temporary files regularly

### 3. Code Maintenance
1. Version control your source code
2. Keep separate directories for different experiments
3. Document parameter changes and their effects
4. Archive successful run configurations

## Support and Resources

### CHPC Documentation
- [CHPC User Guide](https://www.chpc.ac.za/documentation/)
- [Lengau Cluster Documentation](https://www.chpc.ac.za/documentation/lengau/)
- [PBS Job Scheduler Guide](https://www.chpc.ac.za/documentation/lengau/job-scheduling/)

### Contact Information
- **CHPC Support:** support@chpc.ac.za
- **Technical Issues:** Submit tickets through CHPC portal
- **Account Issues:** Contact your project administrator

### Additional Resources
- Intel Parallel Studio XE documentation
- MPI programming guides
- High-performance computing best practices

## Version History

- **v1.0** - Initial setup with Intel toolchain
- **v1.1** - Fixed module conflicts and compilation issues
- **v1.2** - Added OpenMP support and optimization flags
- **v1.3** - Comprehensive troubleshooting guide

---

**Last Updated:** October 2025  
**Maintainer:** msovara@lengau.chpc.ac.za  
**Project:** ASTR1245 - Vela Supernova Remnant Analysis
