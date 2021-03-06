# New stuff in __bsseq__
Peter Hickey  
24 June 2016  



## __bsseq__ in 1 slide

> A collection of tools for analyzing and visualizing bisulfite sequencing data

### Analysis pipeline


```r
library(bsseq)
bsseq <- read.bismark(files, sampleNames) # BSseq object
bsseq <- BSmooth(bsseq) # BSseq object
bsseq_tstat <- BSmooth.tstat(bsseq, group1, group2) # BSseqTstat object
dmrs <- dmrFinder(bsseq_tstat) # data.frame object
# Permutations (not shown)

plotManyRegions(bsseq, regions = dmrs)
```
## Problem 1: Limited to 2-group (pairwise) comparisons {.build}

New dataset, $n = 45$ (6 people $\times$ 8 conditions) means `choose(8, 2)` = 28 pairwise comparisons.

### Solution: F-stat


```r
# Instead of bsseq_tstat <- BSmooth.tstat(bsseq, group1, group2)
bsseq_fstat <- BSmooth.fstat(bsseq, design, contrasts) # BSseqStat object
bsseq_fstat <- smoothSds(bsseq_fstat)
bsseq_fstat <- computeStat(bsseq_fstat, coef = coef)
dmrs <- dmrFinder(bsseq_fstat)
# Permutations (not shown)

# Alternatively, do the above in one hit (not yet exported)
bsseq:::fstat.pipeline(bsseq, design, contrasts, nperm = 1000, coef = NULL)
```

- Built on __limma__

## Problem 2: Smoothed _BSseq_ object is painfully big


```r
dim(bsseq)
#> [1] 28220561       45
pryr::object_size(bsseq)
#> 30.7 GB
```

### Solution: _HDF5Array_ package by Hervé Pages

__Work in progress__


```r
library(HDF5Array)
dim(bsseq_hdf5)
#> [1] 28220561       45
pryr::object_size(bsseq_hdf5)
#> 226 MB
```

## It's not a free lunch


```r
# Example operation that realises result in memory
system.time(max(assay(bsseq, "Cov")))
#> user  system elapsed
#> 6.353   0.001   6.356
system.time(max(assay(bsseq_hdf5, "Cov")))
#> user  system elapsed
#> 522.058   3.278 525.448
#> bsseq_hdf5_scatch uses local rather than networked disk to store HDF5 files
system.time(max(assay(bsseq_hdf5_scratch, "Cov")))
#> user  system elapsed
#> 494.708   1.542 496.349
```

## But it's pretty cheap and damn tasty

_HDF5Array_ uses __delayed operations__ that operate chunkwise


```r
# Example of a delayed operation operation
system.time(getMeth(bsseq, type = "smooth"))
#> user  system elapsed
#> 124.877  15.651 140.556
# Nothing computed because operation is delayed until object is realised!
system.time(getMeth(bsseq_hdf5, type = "smooth"))
# user  system elapsed
#> 0.001   0.000   0.001
#> Realise the result as an array in memory (alternatively, writeHDF5Dataset())
system.time(as.array(getMeth(bsseq_hdf5, type = "smooth")))
#> user  system elapsed
#> 237.207  19.873 257.133
system.time(as.array(getMeth(bsseq_hdf5_scratch)))
#> user  system elapsed
#> 236.539  19.349 255.940
```
