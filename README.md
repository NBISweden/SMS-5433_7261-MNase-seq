# 5433_read_processing_pipelines

Pipelines for processing reads from ChIP-seq or MNase-seq experiments with focus on subtelomeric and other repetitive regions.




## Workflow description

Two workflows are included, which only differ in their starting point:

* a directory with `fastq` files

* a list of SRA accessions.


The entire workflow runs within a [conda](https://docs.conda.io/en/latest/) environment. The only prerequisite is havng a working internet connection for downloading files from the SRA repository, if required. Project specific variables (paths, adapter sequences) are provided via a configuration file `config.yml`.

The pipelines are developed in [Snakemake](https://snakemake.readthedocs.io/en/stable/) workflow management tool.

The starting point for the pipeline are appropriately formatted lists of SRA accessions, which are downloaded from SRA. They are subsequently processed by:

1. Adapters and low quality bases are trimmed off the 3' of the reads;

2. QC is performed before and after adapter trimming;

3. Reads are mapped to the reference genome using bowtie;

4. Normalised tracks (to 1x coverage) are computed.

Reference genome is built as part of the workflow, however the fasta files used for its creation need to be provdided (i.e. downloaded earlier and their path provided via the configuration file).


## Installation and running

As this pipeline runs within a conda environment, you need to install conda first, create the environmnet and then start the pipeline. Small differences between local and cluster usage are covered below. 

Please note that conda is not the best solution for reproducible research, it is a convenient package management tool which offers a possibility to run analyses in defined environments. The build of these environments depends on the local setup, hence it may happen that under some circumstances it is not possible to exactly reproduce given environment.


### Conda installation

There are several `conda` distributions and modifications available.
The simplest is to install [Miniconda](https://docs.conda.io/en/latest/miniconda.html). To do this download the installer for your system and follow the installations instructions.


Note on versions: It is entirely up to you whether you use `Miniconda 2` or `3`. The desired version of `python` can be installed inside the environment.


After the installation you need to configure `conda` to be able to install all necessary packages:

```
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
```

Check if the channels have been added correctly:

```
conda config --show channels
```

should result in:
```
channels:
  - conda-forge
  - bioconda
  - defaults
```



Note: On Rackham you can also use the conda module:

```
module load bioinfo-tools
module load conda/latest
```

Note2: On Dardel it is best to install conda in own home directory, as described in conda documentation. The enviroment data can (and should) be kept in project directories.



### Conda environment

Now that you have conda installed, you can create the environment and install packages. More on [conda environments](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html).


```
conda env create -f environment.yml -n dataproc
```

Note: It is easier to install packages using the file `environment.yml` which lists the packages and versions. Sometimes however, this fails (i.e. seems to be stuck for a long time at the step "Solving environment"), so one solution is to create an empty environment, activate it and install packages. These will be saved in the environment, so you only do it once (i.e. **not** for each session).

```
# first we create an environment with mamba, a faster package manager than conda
conda create -n mamba mamba

# we activate the environment
conda activate mamba

# now we create the environment DataProc2 which will be our working environment
mamba create --no-default-packages -p /proj/snic2020-16-205/private/nbis5433/conda/DataProc2 -c bioconda snakemake=5.21.0 bowtie=1.2.3 cutadapt=3.0 sra-tools=2.10.1 fastqc=0.11.9 samtools=1.10 multiqc=1.9 deeptools=3.5.0

# we can activate the working environment
conda activate /proj/snic2020-16-205/private/nbis5433/conda/DataProc2


```

Please substitute the directory path with currently used system and allocation specifics.


### Activating conda environment

```
conda activate /proj/snic2020-16-205/private/nbis5433/conda/DataProc2
```

Sometimes you need to perform this step prior to environment activation if the shell gives you a messagae that you need to configure it for using `conda activate`:

```
source ~/miniconda2/etc/profile.d/conda.sh
conda activate /proj/snic2020-16-205/private/nbis5433/conda/DataProc2
```




### Pipeline installation

Firstly, you need `git` on your computer. A tutorial on how to install it can be found [here](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) or [here](https://github.com/git-guides/install-git).


To install the pipeline:


```
cd /proj/dir

git clone https://github.com/agata-sm/5433_read_processing_pipelines.git

cd 5433_read_processing_pipelines
```

You may want to inspect the contents:

```
ls
PE		README.html	README.md	SE		environment.yml
```

The pipelines for SE and PE reads are in their respective directories:

```
ls PE/
Snakefile_procdata_PE_v.0.2   accessions_PE.txt     submit-snakemake-sra.sh
Snakefile_procdata_fastq_PE   cluster.yml       submit-snakemake.sh
Snakefile_procdata_fastq_PE_v.0.1 config.yml
```

**Fastq pipeline**

* `Snakefile_procdata_fastq_PE` the snakefile which is the pipeline code;

* `submit-snakemake.sh` is used to run the pipeline on a cluster, where each step is run as a separate job managed by the cluster's queueing manager `SLURM`.


**SRA pipeline**

* `Snakefile_procdata_PE_v.0.2` the snakefile which is the pipeline code;

* `accessions_PE.txt` tab-delimited file with SRA accessions to process, their dataset name and sequencing layout (SE, PE):

```
accession_sra	dataset	layout
SRR771478	Chromatin_remodellers_MNaseseq_PRJNA192582	PE
SRR771479	Chromatin_remodellers_MNaseseq_PRJNA192582	PE
SRR771483	Chromatin_remodellers_MNaseseq_PRJNA192582	PE
SRR771486	Chromatin_remodellers_MNaseseq_PRJNA192582	PE
```

You can list several datasets in one accession file, and the results will be saved in the directories named after the respective dataset.

* `submit-snakemake-sra.sh` is used to run the pipeline on a cluster, where each step is run as a separate job managed by the cluster's queueing manager `SLURM`.

OBS! File `Snakefile_procdata_fastq_PE_v.0.1` is the older version, now deprecated. It will be deleted in the future.


**Config files**


* `config.yml` provides project specific information such as paths to results, logs, genome.fa etc.

Please note that by default `Snakemake` saves results files relative to its working directory. You can change this behaviour by providing the full path to output directories in `config.yml`


* `cluster.yml` contains information relevant to executing this workflow on a cluster (Rackham) such as project accession, number of cores and wall time for each step.



### Running on Rackham

The pipeline main process runs in the background and controls jobs which are sent to the `SLURM` queue. If you log out of Rackham, the process will end and pipline execution will be halted. To prevent this, you can use a small program called `screen` which allows you to detach from the shell without disrupting the process running in it.

First, start the program by typing:

```
screen 
```

A new terminal appears. You can start a process in it, disconnect from it, then reconnect at any time.

To start a new screen press `Ctrl-a` (together), then `c`. A new shell appears, and you detach from it later without progress loss.

Type:

```
module load bioinfo-tools
module load conda/latest

source ~/miniconda2/etc/profile.d/conda.sh

conda activate /proj/snic2020-16-205/private/nbis5433/conda/DataProc2

bash submit-snakemake.sh
```

After some waiting, `Snakemake` starts displaying messages on steps being performed. You can now detach:

Press `Ctrl-a` (together), then `d`.

To reconnect type `screen -r`


### Running locally

You can use `screen` as above, but you do not necessarily have to do it.


You activate the conda environment:

```
source ~/miniconda2/etc/profile.d/conda.sh

conda activate DataProc2

snakemake --snakefile Snakefile_procdata_SE_v.0.4 --cores 1
```


### Monitoring the pipeline run


To monitor jobs in the queue (on Rackham):

```
jobinfo -u <userid>
```

To monitor pipeline progress:

identify the most recent log file:

```
ll .snakemake/log
```

and view its contents

```
.snakemake/log/NNN.snakemake.log
```


If the pipeline is interrupted and throws an error message about directory being locked on attempts to resume the run:

```
snakemake --snakefile Snakefile_procdata_SE_v.0.4 -n --unlock --cores 1
```




### How I run it on Rackham

This is a sequence of commands I used when running the pipeline. I have `Miniconda 2` installed in my home directory.

For environment creation (I created the environment in the project directory, so you can use it too, no need to recreate it, but if you'd like to give it a go, you can rename it):

```
conda create -n mamba mamba

conda activate mamba

mamba create --no-default-packages -p /proj/snic2020-16-205/private/nbis5433/conda/DataProc2 -c bioconda snakemake=5.21.0 bowtie=1.2.3 cutadapt=3.0 sra-tools=2.10.1 fastqc=0.11.9 samtools=1.10 multiqc=1.9 deeptools=3.5.0

conda activate /proj/snic2020-16-205/private/nbis5433/conda/DataProc2
```

Pipeline run:

```
screen

Ctrl-a
c

module load bioinfo-tools
module load conda/latest

source ~/miniconda2/etc/profile.d/conda.sh

conda activate /proj/snic2020-16-205/private/nbis5433/conda/DataProc2

bash submit-snakemake.sh

Ctrl-a
d
```

To monitor jobs in the queue:

```
jobinfo -u agata
```

To monitor pipeline progress:

identify the most recent log file:

```
ll .snakemake/log
```

and view its contents

```
.snakemake/log/NNN.snakemake.log
```


## Next steps

After the run is completed, you may want to generate more tracks, for example tracks which would compare ChIP to input, or average of replicates. This can be done using 
[deepTools](https://deeptools.readthedocs.io/en/develop/).

A tool of interest is
[bigwigCompare](https://deeptools.readthedocs.io/en/develop/content/tools/bigwigCompare.html). 
This tool compares two bigWig files based on the signal in each bin. This output value can be the ratio of the number of reads per bin, the log2 of the ratio, the sum or the difference.

deepTools are included in the conda environment, so all you need to do is to use a command to perform the task you need. For example, to compute log2 ratio of `chip_rep1.bw`  to `input_rep1.bw` with bin size 1 bp (a very small bin size, we use it for *S. cerevisiae*, but be aware that for larger genomes this should only be done over a small region):

```
bigwigCompare --binSize 1 --numberOfProcessors 1 --outFileName chip_vs_input.log2.bw --bigwig1 chip_rep1.bw --bigwig2 input_rep1.bw --operation log2 --pseudocount 0.01 --skipZeroOverZero --skipNonCoveredRegions 
```

In this example we add `--pseudocount 0.01` to avoid division by zero.
