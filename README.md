# README

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.4741501.svg)](https://doi.org/10.5281/zenodo.4741501)  

**This pipeline has been replaced with MetaWorks: A flexible, scalable bioinformatic pipeline for multi-marker biodiversity assessments available from https://github.com/terrimporter/MetaWorks**

This repository processes rbcL metabarcodes. **SCVURL** refers to the programs, algorithms, and reference datasets used in this data flow: **S**EQPREP, **C**UTADAPT, **V**SEARCH, **U**NOISE, **R**bc**L** classifier. 

The pipeline begins with raw paired-end Illumina MiSeq fastq.gz files. Reads are paired. Primers are trimmed. All the samples are pooled for a global analysis. Reads are dereplicated and denoised producing a reference set of exact sequence variants (ESVs). ESVs are filtered again by retaining only the longest open reading frames (ORFs) and removing outliers (ORFs unusually sort or long).  These ORFs are taxonomically assigned using the rbcL cpDNA reference set available from https://github.com/terrimporter/rbcLClassifier and is used with the RDP Classifier (Wang et al., 2007) available from https://sourceforge.net/projects/rdp-classifier/ .

This data flow has been developed using a conda environment and snakemake pipeline for improved reproducibility. It will be updated on a regular basis so check for the latest version at https://github.com/terrimporter/SCVURL_rbcL_metabarcode_pipeline/releases .

## Outline

[How to cite](#How-to-cite)  

[Standard pipeline](#standard-pipeline)  

[Implementation notes](#implementation-notes)  

[References](#references)  

[Acknowledgements](#acknowledgements)  

## How to cite

You can cite the publication:  
Maitland, V. C., Robinson, C. V., Porter, T. M., & Hajibabaei, M. (2020). Freshwater diatom biomonitoring through benthic kick-net metabarcoding. PLOS ONE, 15(11), e0242143. doi: 10.1371/journal.pone.0242143  

You can also cite this repository directly:  
Teresita M. Porter. (2020, January 14). SCVURL RbcL Metabarcode Pipeline (Version v1.0.4). Zenodo. http://doi.org/10.5281/zenodo.4741501   

## Standard pipeline

### Overview of the standard pipeline

If you are comfortable reading code, read through the snakefile to see how the pipeline runs as well as which programs and versions are used.  Otherwise you can just list all the programs in the conda environment, see [Implementation notes](#implementation-notes) below.  

#### A brief overview:

Raw paired-end reads are merged using SEQPREP v1.3.2 from bioconda (St. John, 2016).  This step looks for a minimum Phred quality score of 20 in the overlap region, requires at least 25bp overlap.

Primers are trimmed in two steps using CUTADAPT v2.6 from bioconda (Martin, 2011).  This step looks for a minimum Phred quality score of at least 20 at the ends, forward primer is trimmed first based on its sequence, no more than 3 N's allowed, trimmed reads need to be at least 150 bp, untrimmed reads are discarded.  The output from the first step, is used as in put for the second step.  This step looks for a minimum Phred quality score of at least 20 at the ends, the reverse primer is trimmed based on its sequence, no more than 3 N's allowed, trimmed reads need to be at least 250 bp, untrimmed reads are discarded.

Files are reformatted and samples are combined for a global analysis.

Reads are dereplicated (only unique sequences are retained) using VSEARCH v2.14.1 from bioconda (Rognes et al., 2016).

Denoised exact sequence variants (ESVs) are generated using VSEARCH with the unoise3 algorithm (Edgar, 2016).  This step removes any PhiX contamination, sequences with predicted errors, and rare sequences.  This step produces zero-radius OTUs (Zotus) also referred to commonly as amplicon sequence variants (ASVs), ESVs, or 100% operational taxonomic unit (OTU) clusters.  Here, we define rare sequences to be sequence clusters containing only one or two reads (singletons and doubletons) and these are removed as 'noise'.  Putative chimeric sequences are then removed using VSEARCH.

The ESVs are translated into every possible open reading frame (ORF) for the plus strand.  The longest ORFs are retained for each ESV and outliers are removed.  Outliers are identified as ORFs with lengths outside the range of the 25th percentile - 1.5\*IQR and the 75th percentile + 1.5\*IQR (IQR, inter quartile range).  This method should help to screen out the most obvious pseudogenes that may have a shorter than expected length due to sequence errors, deletions, and frameshifts, or longer than expected length due to insertions.  There is no guarantee that genuine coding sequences are not erroneously removed during this step.  If your dataset contains taxa with coding sequences known to be unusually shorter or longer than usual, then this filtering step should be ommitted from the pipeline and the ESV table and taxonomic assignments should be based on the cat.denoised.nonchimeras file directly, edit the snakemake file as follows:

```linux
rule create_ESV_table:
    input:
        db=chimera_out
...
rule taxonomic_assignment:
    input:
        chimera_out
```

An ESV table that tracks read number for each ESV (longest open reading frame) sample is generated with VSEARCH.  The command --search_exact is used instead of --usearch_global with --id 1.0 because search_exact is faster and optimized for finding exact matches.

rbcL cpDNA taxonomic assignments are made using the Ribosomal Database classifier v2.12 (RDP classifier) available from https://sourceforge.net/projects/rdp-classifier/ (Wang et al., 2007) using the rbcL eukaryote classifier v1 reference dataset available from https://github.com/terrimporter/rbcLClassifier or the rbcL diatom classifier v1 reference dataset available from https://github.com/terrimporter/rbcLdiatomClassifier .

The final output, rdp.csv, is reformatted to add read numbers for each sample and column headers to improve readability.  Read counts for each ESV and each sample are provided in addition to taxonomic assignments with bootstrap support values.  rdp.csv can be read into R, filtered, reformatted, and reshaped to create an ESV x sample matrix filled with read counts for standard biodiversity analyses.

### Prepare your environment to run the pipeline

1. This pipeline includes a conda environment that provides most of the programs needed to run this pipeline (SNAKEMAKE, SEQPREP, CUTADAPT, VSEARCH, etc.).

```linux
# Create the environment from the provided environment.yml file
conda env create -f environment.yml

# Activate the environment
conda activate myenv.3
```

2. The pipeline requires ORFfinder 0.4.3 available from the NCBI at ftp://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/ORFfinder/linux-i64/ .  This program should be downloaded, made executable, and put in your path.

```linux
# download
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/TOOLS/ORFfinder/linux-i64/ORFfinder.gz

# decompress
gunzip ORFfinder.gz

# make executable
chmod 755 ORFfinder

# put in your PATH (ex. ~/bin)
mv ORFfinder ~/bin/.
```

3. The pipeline also requires the RDP classifier for the taxonomic assignment step.  Although the RDP classifier v2.2 is available through conda, a newer v2.12 is available form SourceForge at https://sourceforge.net/projects/rdp-classifier/ .  Download it and take note of where the classifier.jar file is as this needs to be added to config.yaml .

The RDP classifier comes with the training sets to classify 16S, fungal ITS and fungal LSU rDNA sequences.  To classify rbcL cpDNA sequences, obtain the rbcL Classifier v1 reference set from GitHub 
https://github.com/terrimporter/rbcLClassifier .  Take note of where the rRNAclassifier.properties file is as this needs to be added to the config.yaml .

```linux
RDP:
    jar: "/path/to/rdp_classifier_2.12/dist/classifier.jar"
    t: "/path/to/rbcLClassifier/v1/mydata/mydata_trained/rRNAClassifier.properties"
```

4. In most cases, your raw paired-end Illumina reads can go into a directory called 'data' which should be placed in the same directory as the other files that come with this pipeline.

```linux
# Create a new directory to hold your raw data
mkdir data
```

5. Please go through the config.yaml file and edit directory names, filename patterns, etc. as necessary to work with your filenames.

6. Be sure to edit the first line of each Perl script (shebang) in the perl_scripts directory to point to where Perl is installed.

```linux
# The usual shebang if you already have Perl installed
#!/usr/bin/perl

# Alternate shebang if you want to run perl using the conda environment (edit this)
#!/path/to/miniconda3/envs/myenv.3/bin/perl
```

### Run the standard pipeline

Run snakemake by indicating the number of jobs or cores that are available to run the whole pipeline.  

```linux
# Choose and/or edit the appropriate configuration file (ex. config.yaml)
snakemake --jobs 24 --snakefile snakefile --configfile config.yaml
```

When you are done, deactivate the conda environment:

```linux
conda deactivate
```

## Implementation notes

### Installing Conda and Snakemake

Conda is an open source package and envirobnment management system.  Miniconda is a lightweight version of conda that only contains conda, python, and their dependencies.  Using conda and the environment.yml file provided here can help get all the necessary programs in one place to run this pipeline.  Snakemake is a Python-based workflow management tool meant to define the rules for running this bioinformatic pipeline.  There is no need to edit the snakefile or snakefile_alt files directly.  Changes to select parameters can be made in the config.yaml pipeline.  If you install Conda and activate the environment provided, then you will also get the correct versions of the open source programs used in this pipeline including Snakemake v3.13.3.

Install miniconda as follows:

```linux
# Download miniconda3
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh

# Install miniconda3
sh Miniconda3-latest-Linux-x86_64.sh

# Add conda to your PATH, ex. to ~/bin
cd ~/bin
ln -s miniconda3/bin/conda conda
```

### Check program versions

Ensure that the correct programs from the environment are being used.

```linux
# create conda environment from file
conda env create -f environment.yml

# activate the environment
conda activate myenv.3

# list all programs available in the environment at once
conda list > programs.list

# or, inidivdually check that key programs in the conda environment are being used
which SeqPrep
which cutadapt
which vsearch
which perl

# then, check their version numbers one at a time
cutadapt --version
vsearch --version
```

Version numbers are also tracked in the snakefile.

### Batch renaming of files

Sometimes it necessary to rename raw data files in batches.  I use Perl-rename (Gergely, 2018) that is available at https://github.com/subogero/rename not linux rename.  I prefer the Perl implementation so that you can easily use regular expressions.  I first run the command with the -n flag so you can review the changes without making any actual changes.  If you're happy with the results, re-run without the -n flag.

```linux
rename -n 's/PATTERN/NEW PATTERN/g' *.gz
```

### Symbolic links

Symbolic links are like shortcuts or aliases that can also be placed in your ~/bin directory that point to files or programs that reside elsewhere on your system.  So long as those scripts are executable (e.x. chmod 755 script.plx) then the shortcut will also be executable without having to type out the complete path or copy and pasting the script into the current directory.

```linux
ln -s /path/to/target/directory shortcutName
ln -s /path/to/target/directory fileName
ln -s /path/to/script/script.sh commandName
```

## References

Edgar, R. C. (2016). UNOISE2: improved error-correction for Illumina 16S and ITS amplicon sequencing. BioRxiv. doi:10.1101/081257 

Gergely, S. (2018, January). Perl-rename. Retrieved from https://github.com/subogero/rename  

Maitland, V. C., Robinson, C. V., Porter, T. M., & Hajibabaei, M. (2020). Freshwater diatom biomonitoring through benthic kick-net metabarcoding. PLOS ONE, 15(11), e0242143. doi: 10.1371/journal.pone.0242143  

Martin, M. (2011). Cutadapt removes adapter sequences from high-throughput sequencing reads. EMBnet. Journal, 17(1), pp–10. 

Rognes, T., Flouri, T., Nichols, B., Quince, C., & Mahé, F. (2016). VSEARCH: a versatile open source tool for metagenomics. PeerJ, 4, e2584. doi:10.7717/peerj.2584  

St. John, J. (2016, Downloaded). SeqPrep. Retrieved from https://github.com/jstjohn/SeqPrep/releases  

Tange, O. (2011). GNU Parallel - The Command-Line Power Tool. ;;Login: The USENIX Magazine, February, 42–47.  

Wang, Q., Garrity, G. M., Tiedje, J. M., & Cole, J. R. (2007). Naive Bayesian Classifier for Rapid Assignment of rRNA Sequences into the New Bacterial Taxonomy. Applied and Environmental Microbiology, 73(16), 5261–5267. doi:10.1128/AEM.00062-07  

## Acknowledgements

I would like to acknowedge funding from the Canadian government through the Genomics Research and Development Initiative (GRDI), Metagenomics-Based Ecosystem Biomonitoring (Ecobiomics) project.

Last updated: May 6, 2021
