# Truvari Data

Investigating when two SVs are the same

This repository contains the workflow and notes for creating most of the data for this research

Input/output results can be found at: https://tinyurl.com/TruvariRawData

# Setup
## Requirements

Assembly mapping and variant calling
- [bcftools](http://www.htslib.org/)
- [vcftools](https://vcftools.github.io/index.html)
- [truvari](https://github.com/spiralgenetics/truvari)
- [minimap2/paftools](https://github.com/lh3/minimap2)

Short-read SV discovery and genotyping
- [biograph](https://github.com/spiralgenetics/biograph)
- [manta](https://github.com/Illumina/manta)
- [paragraph](https://github.com/ACEnglish/paragraph)

## Repo structure
- scripts - Steps of the pipeline
- notebooks - Analysis notebooks for plot generation
- metadata - Mainly the sample metadata file
- parallel_helpers - Scripts that helped me parallelize the steps
- stats - Quick analysis joblib dataframes

## Genome References

Any reference genome can be used. We chose to subset references to only Autosomes and X/Y.
References need to be indexed using [`bwa index`](https://github.com/lh3/bwa). We have run
on hg19, grch38, chm13, and pr1. PR1 is not presented in the publication.

# Pipeline

Below are the steps of the pipeline to create the data and generate summaries.

## Download haplotype fastas

The haplotype fasta files are pulled from their public locations using 
```bash
bash eichler_download.sh
bash li_download.sh
```

## Assembly Stats:

Calculate the basic assembly stats with [calN50](https://github.com/lh3/calN50). This summary is inside the repository's `stats/asm_fastastat.jl`. An example `files.txt` is available in `metadata/assembly_fasta_meta.txt`
```bash
python n50_summary.py files.txt asm_fastastat.jl
```  

## Haplotype mapping and variant calling
Map each haplotype to a reference and call variants using mapping.sh

```bash
bash mapping.sh haplotype.1.fasta sample.hap1 reference.fasta
```

This creates an alignment file `sample.hap1.paf`, a vcf file `sample.hap1.var.vcf.gz`, and an index for that vcf.
A helper script `parallel_helpers/make_all_mapping_jobs.sh` can make per-sample bash scripts.

## Mapping Stats
If you collect per-haplotype mapping logs from `mapping.sh`, you can build a table from all the stats generated.
See consolidate_mapping_stats.py for details. This summary is also inside `stats/asm_mapstat.jl`

## Calculating assembly coverage per-haplotype
Given the paf files, create a coverage bed for each haplotype

```bash
bedtools genomecov -i <(cut -f6,8,9 sample.hap1.paf | bedtools sort) -g reference.fa.fai -bga > sample.hap1.cov.bed
```

## Exact Intra-merge
Merge the two haplotype vcf files together using
  
```bash
bash merge_haps.sh sample.hap1.var.vcf.gz sample.hap2.var.vcf.gz sample output/path/var.vcf.gz
```

This creates a diploid vcf file name `sample.vcf.gz` and its index. This is the start of
our 'exact' merge VCFs. At this point, the files should be orgainzed in the below file
structure, thus the 4th argument 'output/path/var.vcf.gz'

## File structure

The per-sample vcfs are organized into sub-directories starting with the exact vcf path
`data/reference/project/sample/merge_strategy.vcf.gz` (e.g. `data/intra_merge/chm13/li/NA12878/exact.vcf.gz`)
Be sure to create the index for the VCFs, also

## Strict/Loose Intra-merge
Run truvari to collapse the exact intra-merge vcfs 

```bash
bash single_sample_collapse.sh $exact_vcf_path reference.fa
```
Where `$exact_vcf_path` is described above in File Structure.

This will run `truvari collapse` for strict and for loose merging and place them
in the appropriate directory. For example `data/chm13/li/NA12878/strict.vcf.gz` 

Each collapse also produces `removed.strict.vcf.gz`, vcf indexes, and logs in `collapse.strict.log`

## Annotate with assembly coverage
Run the following to make a new VCF that has assembly coverage information

```bash
python intra_hc_region_annotate.py input.vcf.gz sample.hap1.cov.bed sample.hap2.cov.bed | bgzip > output.vcf.gz
```
After running, I replace input.vcf.gz with output.vcf.gz

## Make single sample summary stats

Beside a VCF, create a joblib DataFrame of just SVs >= 50bp using, for example

```bash
bash make_stats.sh data/chm13/exact.vcf.gz
```

Once these are all created, run
```bash
python consolidate_intra_stats.py data/intra_merge single_sample_stats.jl
```

This will look for all subdirectories that have subdirectories that have subdirectories
containing '.jl' files. (a.k.a. reference/project/sample/merge_strategy.jl)
So that it can append columns of those path metadata information to each row, drop the
index, and make a usable dataframe for the SVCharacteristics notebook.

## Inter-sample merging

We now merge between samples. This needs to have a specific file-structure, also.
In a directory,  (e.g. `data/inter_merge`), we will create all the combinations of 
merges. For a single reference, we need to take all the exact 
merges and then do an exact merge to create `exact.exact.vcf.gz`

Do this by calling
```bash
python multi_merge.py data/intra_merge data/inter_merge/chm13 data/references/chm13.fa > the_merge_script.sh
```

This will create all the commands needed to make all combinations of merges for a single reference.
Repeat for each reference.

## Inter-merge stats

Once all of the samples have been processed, create the stats via
```bash
bash make_stats.sh grch38/exact/exact.vcf.gz
```

Consolidate the stats with 
```bash
python consolidate_inter_stats.py *.jl
```
where  `*.jl` are all from the `intra_merge` made during the multi-sample merge step. Note that 
this assumes that `out_dir` is the name of a reference.

## Build the paragraph multi-sample VCFs

Given one of the multi-sample merges, create a paragraph reference out of only the svs using
```bash
 bash run_paragraphs.sh data/multi_merge/grch38/strict/strict.vcf.gz \
 						data/reference/grch38/grch38.fa \
						data/paragraph_ref
```

This will try to run paragraph just enough to make the reference inputs needed to run a
modified version of paragraph https://github.com/ACEnglish/paragraph

## Other merges

To compare other merging methods, we made a custom script `naive_50.py` to do 50%
reciprocal overlap merging. We then downloaded three other SV merging tools to compare.

- [SURVIVOR v1.0.6](https://github.com/fritzsedlazeck/SURVIVOR)
- [Jasmine v1.1](https://github.com/mkirsche/Jasmine)

Using the strict intra-sample merge SVs, we ran each program with default parameters.  
First, collect the full paths to each of the vcfs you want to merge via
```bash
find /full/path/to/data/intra_merge/chm13/ -name "strict.vcf.gz"  > input_files.txt
```

Next, edit `perform_other_merges.sh` so that each of the programs' executable are pointed to
in their correct variable. 
Run this script inside of the working directory e.g. `data/other_merges/grch38/`.
```bash
bash perform_other_merges.sh input_files.txt
```
Note this creates a lot of temporay files to reformat things to accommodate these programs.
All the temporary files are stored in the `pwd`/temp and can be removed without affecting downstream
steps of this pipeline.

Additionally, to facilitate the notebook summary of these files, you can soft-link the joblibs of the truvari
and exact runs from the earlier inter_merge: e.g. `ln -s data/inter_merge/grch38/strict/strict.jl truvari.S.jl`

## Short-read SV discovery

Used existing Manta calls  
Ran BioGraph v7.0  
Calls organized into
`data/short_read_calls/discovery/[biograph|manta]/[sample].[reference].vcf.gz`
Note manta was only run on grch38, so reference==sv

## Short-read SV genoyping

Ran Paragraph and BioGraph v7.0
BioGraph results are in `data/short_read_calls/genotyping/biograph/[sample]/[reference].results.vcf.gz`
Paragraph results are in `data/short_read_calls/genotyping/paragraph/[sample].vcf.gz`
Note only chm13 and grch38 are available.


## Create annotated SV Only VCFs

How did you do this? trf and numneigh and only_svs.py, that's somewhere.

## Benchmarking short-read discovery

Run the script with 
```
bash run_truvari_bench.sh data/ benchmarking_output/ data/short_read_calls/discovery/manta/*.vcf.gz
```
Where
```
Arg1 - the base directory of the data
Arg2 - output destination directory
Arg* - All the remaining args are the discovery VCFs.
```
This assumes that the vcfs have the structure of `<sample>.<reference>.vcf.gz`

## Benchmarking short-read genotyping

Make the base vcf genotypes data from the paragraph sv_only.vcf.gz
"""
python multi_sample_vcf_to_df.py data/paragraph_ref/sv_only.vcf.gz data/paragraph_ref/sv_only.df
"""

Turn the paragraph results into a single dataframe 
"""
python genotype_consolidate.py data/paragraph_ref/sv_only.df paragraph \
                 data/short_read_calls/genotyping/paragraph/results.jl \
				data/short_read_calls/genotyping/paragraph/*.vcf.gz
"""
Run the same thing for biograph

## Genes

Where did the GRCH38_CMRG bed come from.

For each of the other merges' VCFs, run bpovl, turn vcf into dataframe, and do the annotation consolidation join
```
for i in *.vcf.gz;
do
truvari anno bpovl -i $i -a ../GRCh38_CMRG_benchmark_gene_coordinates.bed -o ${i%.vcf.gz}.CBGcoords.anno.jl
truvari vcf2df -i ${i} ${i%.vcf.gz}.jl
done
python consolidate_dfs.py
```

Gencode.v35 bed tracks were downloaded from UCSC genome browser. These bed files were merged and expanded using

```
bedtools merge -i input.bed.gz | python bed_expander.py reference/genome.fa.fai | bedtools sort | bgzip > output.bed.gz
tabix output.bed.gz
```
Note the hg19 bed also needed `sed 's/chr//'`.

These bed files can be used by run_truvari to make the recall for gene regions.
