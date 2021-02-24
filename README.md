# Generate fast trees for clustering purposes in SARS-CoV-2

*NOTE*: THIS IS REPO IS IN ACTIVE DEVELOPMENT. LOTS OF CHANGES ARE TO COME.

## Background

There is an ever increasing number of samples included in phylogenetic tree 
building for SARS-CoV-2. There are a number of complications associated with
building high quality trees for cluster dedection in SARS-CoV-2 (addressed
elsewhere [ADD CITATIONS])

## Running the current pipeline

### Requirements

* NextFlow >= 20.10.0.5430 [https://nextflow.io]
* Either:
  * Docker [https://docker.io]
  * Singularity [https://sylabs.io]
  * Conda [https://conda.io]
* [OPTIONALLY] GraphViz [http://www.graphviz.org] --- to visualise the DAG


### Running the test dataset

- With conda:

```bash
nextflow run MDU-PHL/kovid-trees-nf
```

- With docker

```bash
nextflow run MDU-PHL/kovid-trees-nf -profile docker
```

- With singularity

```bash
nextflow run MDU-PHL/kovid-trees-nf -profile singularity
```

## Running with your own dataset

```bash
nextflow run MDU-PHL/kovid-trees-nf --input_aln relative/path/to/alignment.aln
```

## Pipeline details

```
GOALIGN_REPLACE	
    goalign \
        --auto-detect \
        replace \
        -e -s '[^ACTGactg]' -n '-' \
        -t 1 \
        -o test_replace.aln \
        -i test_aln.aln
    
GOALIGN_CLEAN_SEQS	
    goalign \
        --auto-detect \
        clean \
        seqs \
        --cutoff 0.05 \
        -t 1 \
        -o test_clean.aln \
        -i test_replace.aln
    
CLIPKIT	
    clipkit \
        test_clean.aln \
        -m kpic-smart-gap \
        -l \
        -o test_filtered.aln
    
GOALIGN_DEDUP	
    goalign \
        --auto-detect \
        dedup \
        -l test.dedup \
        -t 1 \
        -o test_dedup.aln \
        -i test_filtered.aln
    
GOALIGN_COMPRESS	
    goalign \
        --auto-detect \
        compress \
        -t 1 \
        -o test_compress.aln \
        --weight-out test.weights \
        -i test_dedup.aln
    
FASTTREE	
    OMP_NUM_THREADS=1 \
        fasttree \
        -nosupport -fastest \
        -out test.nwk \
        -nt test_compress.aln
    
GOTREE_RESOLVE	
    gotree \
        resolve \
        -t 1 \
        -i test.nwk \
        -o test_resolve.nwk \
    
RAXMLNG_EVALUATE	
    raxml-ng \
    --evaluate \
    --blmin 0.0000000001 \
    --threads 1 \
    --prefix test \
    --force perf_threads \
    --model GTR+G4 \
    --tree test_resolve.nwk \
    --msa test_compress.aln \
    --site-weights test.weights
    
GOTREE_BRLEN_ROUND	
    gotree \
        brlen \
        round \
        -p 6 \
        -t 1 \
        -i test.raxml.bestTree \
        -o test_brlen_round.nwk
    
GOTREE_COLLAPSE_LENGTH	
    gotree \
        collapse \
        length \
        -l 0 \
        -t 1 \
        -i test_brlen_round.nwk \
        -o test_collapse_length.nwk
    
GOTREE_REPOPULATE	
    gotree \
        repopulate \
        -t 1 \
        -g test.dedup \
        -i test_collapse_length.nwk \
        -o test_repopulate.nwk \
    
NWUTILS_ORDER	
    nw_order \
        test_repopulate.nwk > test_order.nwk
    
NWUTILS_REROOT	
    nw_reroot \
        test_order.nwk \
        ref\|NC_045512.2\| > test_reroot.nwk
```