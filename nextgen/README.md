## DEPRECATION WARNING

This repository has been deprecated in favor of: <a href="https://github.com/SciLifeLab/bcbio-nextgen">https://github.com/SciLifeLab/bcbio-nextgen</a>.

Please use that instead.

## Overview

Python scripts and modules for automated next gen sequencing analysis.
These provide a fully automated pipeline for taking sequencing results
from an Illumina sequencer, converting them to standard Fastq format,
aligning to a reference genome, doing SNP calling, and producing a
summary PDF of results.

The pipeline runs on single multicore machines, in compute clusters
managed by LSF or SGE, or on the Amazon cloud. [This tutorial][o5]
describes running the pipeline on Amazon with [CloudBioLinux][o6] and
[CloudMan][o7].

The scripts are tightly integrated with the [Galaxy][o1]
web-based analysis tool. Tracking of samples occurs via a web based LIMS
system, and processed results are uploading into Galaxy Data Libraries for
researcher access and additional analysis. See the
[installation instructions for the front end][o2] and a
[detailed description of the full system][o3].

###Overview of workflow
<p align="center">
  <img src="https://raw.github.com/guillermo-carrasco/bcbb/develop/nextgen/pipeline_flow.png" height="700" width="500" alt="Pipeline overview"/>
</p>

[o1]: http://galaxy.psu.edu/
[o2]: https://bitbucket.org/galaxy/galaxy-central/wiki/LIMS/nglims
[o3]: http://bcbio.wordpress.com/2011/01/11/next-generation-sequencing-information-management-and-analysis-system-for-galaxy/
[o5]: http://bcbio.wordpress.com/2011/08/19/distributed-exome-analysis-pipeline-with-cloudbiolinux-and-cloudman/
[o6]: http://cloudbiolinux.org
[o7]: http://wiki.g2.bx.psu.edu/Admin/Cloud

Current build status: [![Build Status](https://secure.travis-ci.org/SciLifeLab/bcbb.png)](http://travis-ci.org/SciLifeLab/bcbb)

## Code structure

The main scripts that handle automation of the analysis and storage
are:

* `scripts/illumina_finished_msg.py` -- Sits on the sequencer
  output machine; run via a cron job every hour to check for new
  output runs.

* `scripts/nextgen_analysis_server.py` -- Main server script;
  runs specific servers for top level analysis management, storage, or
  distributed processing of individual analysis steps:

 * Called with `-q toplevel` -- Coordinate sample processing,
   running the full automated analysis and optionally uploading
   results to Galaxy. `illumina_finished_msg.py` and a Galaxy
   graphical front end both send messages on this queue for starting
   processing.

 * Called with no queue arguments -- Run individual steps in the
   analysis pipeline. Start multiple servers on distributed machines
   connected with a shared filesystem to allow scaling on a
   cluster or Amazon EC2.

 * Called with `-q storage` -- Manage long term storage of larger
   files, like qseq and images.

Specify system specific information in configuration files:

* `config/transfer_info.yaml` -- Configuration on the sequencing
  machine, specifying where to check for new sequencing data.
* `config/post_process.yaml` -- Configuration for analysis and
  storage. This contains links to Galaxy, program locations and
  customization for processing algorithms.
* `config/universe_wsgi.ini` -- Variables used from your Galaxy
  server configuration, including RabbitMQ details for communication
  between the sequencing and analysis machines.

Scripts involved in the processing:

* `scripts/automated_initial_analysis.py` -- Drives the high level analysis of
  sequencing lanes based on information specified through the Galaxy LIMS
  system or in a YAML configuration file. Also produces a PDF summary file with
  statistics on alignments, duplicates, GC distribution, quality scores,
  and other metrics of interest.
* `scripts/upload_to_galaxy.py` -- Handles storing and uploading Fastq,
  alignment, analysis and summary files to Galaxy.

## Installation

### Auto-installation

[Here](https://github.com/SciLifeLab/bcbio-nextgen-deploy) you can find a set of scripts to automatically download and set-up the pipeline.
This set of scripts will pull the pipeline code and install on your system the pipeline and all the necessary requirements to have it working. Basically, there
are three installation options available. First, download the scripts:

        git clone https://github.com/SciLifeLab/bcbio-nextgen-deploy.git && cd bcbio-nextgen-deploy

#### Installation in UPPMAX

After downloading the scripts in your UPPMAX account, the following command will install and configure the pipeline, 
as well as run the standard test suite in a batch job:

        python deploy_non_root.py install

To uninstall the pipeline from your account, execute:

        python deploy_non_root.py uninstall


#### Installation within a virtual machine

If you want to perform the installation within a virtual machine, you can do it executing:

        fab -f deploy_on_vm.py install

This will create a virtual machine, install the pipeline and run the tests. You need [vagrant](http://vagrantup.com) and 
[python fabric](http://docs.fabfile.org/en/1.4.3/) to install the pipeline this way.

#### Local installation

To perform a local installation, execute the same command than installing in UPPMAX:

        python deploy_non_root.py install

The script will take care of installing the pipeline properly depending on the environment (UPPMAX or local).

### Notes
You don't need root access to install the pipeline or to run the tests with this scripts, as it is installed in a python virtual environment. However, to install
the pipeline locally, we assume that you have installed all the pipeline requirements in your machine, otherwise you have to install them. Please refer to 
the requirements section below. 


### Configuration

#### File transfer

When the pipeline moves files between machines, e.g. from the sequencing
machine to the analysis machine they are by default transferred using scp.
In post_process.yaml you can specify to use a different protocol by adding
the key

        protocol: a_protocol

where "a_protocol" is one of "scp", "rsync" and "rdiff-backup".

Note that there are some behavioral differences between the protocols:

* scp -- If a file exists on the destination machine, it will not be 
  transferred.

* rsync -- If a file exists on the destination machine, it will be 'resumed',
  so in case there was a loss of connection during a transfer the file will
  be mended. If the file exists but doesn't match the file being transferred
  it will be replaced.

* rdiff-backup -- If a file exists on the destination but there is no rdiff
  data, it will be overwritten. However, if the file is modified on the
  source somehow after the first transfer, the next transfer will simply
  update the file.

### Parallel execution

The pipeline runs in parallel in two different ways:

* multiple cores -- Analyses will run in parallel using multiple cores
  on a single machine. This requires only the `mulitprocessing`
  Python library, included by default with most Python
  installations. Change `num_cores` in your `post_process.yaml` file
  to specify the number of parallel cores to use.

* parallel messaging -- This allows scaling beyond the cores on a
  single machine. It requires multiple machines with a shared
  filesystem, with communication handled using RabbitMQ messaging.
  This is ideally suited for clusters.

To enable parallel messaging:

1. Configure RabbitMQ as described below. Ensure all processing
   machines can talk to the RabbitMQ server on port 5672. Update
   `universe_wsgi.ini` to contain the server details.

2. Change `num_cores` in `post_process.yaml` to `messaging`

The `distributed_nextgen_pipeline.py` helper script runs pipelines in a
distributed cluster environment. It takes care of starting worker
nodes, running the processing, and then cleaning up after jobs:

1. Edit your `post_process.yaml` file to set parameters in the
  `distributed` section corresponding to your environment: this
  includes the type of cluster management, arguments to start jobs,
  and the number of workers to start.

2. Execute `distributed_nextgen_pipeline.py` using the same arguments as
   `automated_initial_analysis.py`: the `post_process.yaml`
   configuration file, the directory of fastq files, and a
   `run_info.yaml` file specifying the fastq details, barcodes,
   and the types of analyses to run.

If you have a different architecture you can run the distributed
processing by hand:

1. Start the processing server, `nextgen_analysis_server.py` on each
   processing machine. This takes one argument, the
   `post_process.yaml` file (which references the `universe_wsgi.ini`
   configuration).

2. Run the analysis script `automated_initial_analysis.py`. This will
   offload parallel work to the workers started in step 3 but also
   does processing itself, so is also submitted as a job in cluster
   environments.

### Testing

The test suite exercises the scripts driving the analysis, so
are a good starting point to ensure correct installation.
Run tests from the main code directory using [nose][i7]:

      nosetests -v -s

`tests/test_automated_analysis.py` exercises the full framework using
an automatically downloaded test dataset. It runs through barcode
deconvolution, alignment and full SNP analysis. Tweak the
configuration for the tests for your environment:

* `tests/data/automated/post_process.yaml` -- May need adjustment to
  point to installed software in non-standard locations. Change the
  num_cores parameter to test multiple processor and parallel
  execution.
* `tests/data/automated/universe_wsgi.ini` -- Defines the location of
  the RabbitMQ server for messaging based parallel execution during
  tests.
* `tests/data/automated/run_info.yaml` -- Change the `analysis` variable
  can to 'Standard' if SNP calling is not required in your
  environment. This will run a smaller pipeline of alignment and analysis.

To run the basic functionality test suite, which takes less time than the full test suite, use:

        nosetests -v -s -a standard

### RabbitMQ messaging server

RabbitMQ messaging manages communication between the sequencing
machine and the analysis machine. This allows complete separation
between all of the machines. The RabbitMQ server can run anywhere; an
easy solution is to install it on the Galaxy and analysis server:

        (yum or apt-get) install rabbitmq-server

Setup rabbitmq for passing Galaxy and processing messages:

        rabbitmqctl add_user <username> <password>
        rabbitmqctl add_vhost bionextgen
        rabbitmqctl set_permissions -p bionextgen <username> ".*" ".*" ".*"

Then adjust the `[galaxy_amqp]` section of your `universe_wsgi.ini`
Galaxy configuration file. An example configuration is available in
the config directory; you'll need to specifically change these three
values:

        [galaxy_amqp]
        host = <host you installed the RabbitMQ server on>
        userid = <username>
        password = <password>

### ssh keys

The sequencing, analysis and storage machines transfer files using
secure copy. This requires that you can securely copy files between
machines without passwords, using [ssh public key authentication][i1].
You want to enable password-less ssh for the following machine
combinations:

* Analysis server to `illumina_finished_msg` machine
* Storage server to `illumina_finished_msg` machine

### Sequencing machines

The sequencer automation has been fully tested using Illumina GAII and
HiSeq sequencing machines. The framework is general and supports other
platforms; we welcome feedback from researchers with different
machines at their institutions.

Illumina machines produce run directories that include the date, machine
identifier, and flowcell ID:

    110216_HWI-EAS264_00035_FC638NPAAXX

A shortened name, with just date and flowcell ID, is used to uniquely
identify each flowcell during processing.

### Development environment

The installation instructions assume that you have full root access to install
python modules and packages (production environment). If this is not the case,
you may want to install a python VirtualEnv and other tools automatically on your $HOME
to ease your development needs using the following script:

        wget https://bitbucket.org/brainstorm/custom_env/raw/1cd4f4ae27d5/pyHost.sh && ./pyHost.sh

[i1]: http://macnugget.org/projects/publickeys/
[i2]: http://cloudbiolinux.org
[i3]: http://www.biomedcentral.com/1471-2105/11/S12/S4
[i4]: https://github.com/chapmanb/cloudbiolinux/blob/master/fabfile.py
[i5]: https://github.com/chapmanb/cloudbiolinux/blob/master/data_fabfile.py
[i6]: https://bitbucket.org/afgane/mi-deployment/src
[i7]: http://somethingaboutorange.com/mrl/projects/nose/

## Requirements

### Next gen analysis

* [Picard][3] (version 1.64)
* [bowtie][5] or [bwa][5b]
* [FastQC][6]
* [GATK][4] (at least version 1.4)
* [snpEff][16] (at least version 2.0.5)
* fastqscreen (version 0.2.1)

[3]: http://picard.sourceforge.net/
[4]: http://www.broadinstitute.org/gsa/wiki/index.php/The_Genome_Analysis_Toolkit
[5]: http://bowtie-bio.sourceforge.net/
[5b]: http://bio-bwa.sourceforge.net/
[6]: http://www.bioinformatics.bbsrc.ac.uk/projects/fastqc/
[16]: http://sourceforge.net/projects/snpeff/

### Processing infrastructure

* RabbitMQ for communication between machines
* LaTeX and pdflatex for report generation

### Optional software for generating report graphs

* R with ggplot2, plyr, sqldf libraries.
* [rpy2][11]. Your R needs to be built with shared libraries
  available: `./configure --enable-R-shlib`.

### Python modules installed with the package

* [biopython][10]
* [pysam][12]
* [mako][13]
* [PyYAML][14]
* [logbook] [17]
* [celery][18]
* [amqplib][15]
* [gdata][19]

[10]: http://biopython.org
[11]: http://rpy.sourceforge.net/rpy2.html
[12]: http://code.google.com/p/pysam/
[13]: http://www.makotemplates.org/
[14]: http://pyyaml.org/
[17]: http://packages.python.org/Logbook
[18]: http://celeryproject.org/
[19]: http://code.google.com/p/gdata-python-client/

## Variant calling

Broad's [GATK][4] pipeline drives variant (SNPs and Indels) analysis. This
requires some associated data files, and also has some configurable options. The
relevant section from the `post_process.yaml` file is:

    coverage_depth: "low" # other options: high
    coverage_interval: "exome" # other options: genome, regional
    dbsnp: variation/dbsnp_132.vcf
    train_hapmap: variation/hapmap_3.3.vcf
    train_1000g_omni: variation/1000G_omni2.5.vcf
    train_indels: variation/Mills_Devine_2hit.indels.vcf

The dbSNP and training files are from the [GATK resource bundle][v1]. These are
inputs into the training models for recalibration. The automated
[CloudBioLinux][o6] data scripts will download and install these in the
variation subdirectory relative to the genome files.

The `coverage_depth` and `coverage_interval` are adjustable from the defaults
within individual `run_info.yaml` files describing the sample; a fastq file
with standard phred quality scores, full genome coverage and high sequencing
depth:

    - analysis: SNP calling
      algorithm:
        quality_format: Standard
        coverage_interval: genome
        coverage_depth: high

[v1]: http://www.broadinstitute.org/gsa/wiki/index.php/GATK_resource_bundle

## Configuration options

The YAML configuration file provides a number of hooks to customize analysis.
Place these under the `analysis` keyword. For variant calling:

- `aligner` Aligner to use: [bwa, bowtie, bowtie2, mosaik, novoalign]
- `trim_reads` Whether to trim off 3' B-only ends from fastq reads [false, true]
- `variantcaller` Variant calling algorithm [gatk, freebayes]
- `quality_format` Quality format of fastq inputs [Illumina, Standard]
- `coverage_interval` Regions covered by sequencing. This influences filtra
- `hybrid_target` BED file with target regions for hybrid selection experiments.
- `variant_regions` BED file of regions to call variants in.
- `ploidy` Ploidy of called reads. Defaults to 2 (diploid).

Global reference files for variant calling and assessment:

- `train_hapmap`, `train_1000g_omni`, `train_indels` Training files for GATK
  variant recalibration.
- `call_background` Background VCF to use for calling.

## Internals: files generated by this pipeline

### Initial Fastq files (pre-analysis)

After basecalling, a number of FastQ files are generated via <a href="https://github.com/brainstorm/bcbb/blob/master/nextgen/scripts/solexa_qseq_to_fastq.py">solexa_qseq_to_fastq.py</a> script:

    1_081227_B45GT6ABXX_1_fastq.txt
    1_081227_B45GT6ABXX_2_fastq.txt
    (...)

The template of them being:

    lane_date_fcid_<1|2>_fastq.txt

Where 1|2 is the forward and reverse read respectively. Only reads that 
pass the quality filter (PF) are kept. Quality scores
are not converted to Sanger, but are kept in the produced format (currently
Illumina 1.3+). The dots in the fastq files are converted into Ns.

## Post-processing generated files

Those are distributed in three directories: alignments, barcode and the top level run directory.

### Top level directory

Those files comprise both plain text files, images and structured data
that can be useful both automatically and in human-readable form to get
different aspects about the sequencing results.

* `run_summary.yaml`

Contains a structured view of the run global parameters.

    - barcode_id: '2'
      barcode_type: SampleSheet
      lane: '8'
      metrics:
        Aligned: '1204752'
        Pair duplicates: '881332'
        Read length: '101'
        Reads: '11594437'
      request: ''
      researcher: ''
      sample: ''

* PDF files

Taking a look at an specific sample in a lane, we find the different
sub-components that conform the final summary report (`*-summary.pdf`).
The summary contains GC content, read quality figures and an insert size
histogram.

    6_110126_B816J0ABXX_5_1_fastq_txt_quality.pdf
    6_110126_B816J0ABXX_5_2_fastq.txt.segments.hist.pdf
    6_110126_B816J0ABXX_5-sort-dup-gc.pdf
    6_110126_B816J0ABXX_5_1_fastq.txt.segments.hist.pdf
    6_110126_B816J0ABXX_5-sort_1_fastq_qual.pdf
    6_110126_B816J0ABXX_5-sort-dup-insert.pdf
    6_110126_B816J0ABXX_5_2_fastq_txt_quality.pdf
    6_110126_B816J0ABXX_5-sort_2_fastq_qual.pdf
    6_110126_B816J0ABXX_5-sort-summary.pdf

* BigWig files

Per-sample wigtoBigWig converted files.

* `*_metrics files`

They contain plain text values from picard, namely: 

    1_110126_B816J0ABXX_5-sort-dup.align_metrics
    1_110126_B816J0ABXX_5-sort-dup.gc_metrics
    1_110126_B816J0ABXX_5-sort-dup.dup_metrics
    1_110126_B816J0ABXX_5-sort-dup.insert_metrics

They contain output metrics from Picard, which are parsed later and
used to generate plots. For instance, for `sort-dup.align_metrics`, the
output from net.sf.picard.analysis.CollectAlignmentSummaryMetrics is
stored.

### alignments directory

Contains the results of the alignments for each sample. As we see
on the listing below, lane 1, barcode id 5 has been aligned in 
[SAM][in1] and BAM formats. For convenience,
to facilitate SNP calling, for instance, a sorted BAM file is also
generated.

    1_110126_B816J0ABXX_5.sam
    1_110126_B816J0ABXX_5.bam
    1_110126_B816J0ABXX_5-sort.bam
    1_110126_B816J0ABXX_5_1_fastq.bam

### *_barcode directories

Those contain fastq files conforming with the naming schema we've seen before. They are the result of the demultiplexing process, where the "unmatched" files contain the reads that have not passed the approximate barcoding matching algorithm:

    4_110126_B816J0ABXX_1_1_fastq.txt
    4_110126_B816J0ABXX_5_2_fastq.txt
    4_110126_B816J0ABXX_1_2_fastq.txt
    4_110126_B816J0ABXX_6_1_fastq.txt
    4_110126_B816J0ABXX_5_1_fastq.txt
    4_110126_B816J0ABXX_6_2_fastq.txt

    4_110126_B816J0ABXX_unmatched_1_fastq.txt
    4_110126_B816J0ABXX_unmatched_2_fastq.txt

    4_110126_B816J0ABXX_bc.metrics
    SampleSheet-barcodes.cfg

`*-barcodes.cfg` contains a simple mapping between barcode id's and the actual barcode sequence:

    3 ATCACGA
    2 ACTTGAA
    9 TAGCTTA
    (...)

The `_bc.metrics` file has a plain read distribution for each barcode:

    2	11594437
    3	20247932
    9	14390566
    unmatched	908420

Barcodes are added to the 3' end of the first sequence. That way, it
remains platform-independent and can be easily handled downstream.
This [GitHub discussion][in2] explains how demultiplexing works. The
demultiplexing is performed by the [barcode_sort_trim.py][in3] script.

[in1]: http://bioinformatics.oxfordjournals.org/content/early/2009/06/08/bioinformatics.btp352.short
[in2]: https://github.com/chapmanb/mgh_projects/commit/3387d82f3496025ad13b69e8d9cbb47cf6ee2af9#nglims_paper/nglims_galaxy.tex-P57
[in3]: https://github.com/brainstorm/bcbb/blob/master/nextgen/scripts/barcode_sort_trim.py

## Upload of barcode read counts to google docs account

Optionally, the read count distribution for each barcode as recorded in the
`*_bc_metrics` files can be uploaded to a spreadsheet  on Google Docs. If there
exists a `gdocs_upload` section in the `post_process.py` configuration file,
the pipeline will attempt  to upload the read counts using the specified
credentials. The `gdocs_upload` section should contain parameters specifying
the  name of the remote spreadsheet on Google Docs and a base64-encoded string
containing the concatenated credentials (on the form  `username:password`) of a
Google account with write permissions:

    gdocs_upload:
        gdocs_dmplx_file: Demultiplex counts
        gdocs_credentials: ZXhhbXBsZUBnbWFpbC5jb206eW91cmdvb2dsZXBhc3N3b3Jk
        gdocs_projects_folder: 'Project read counts'

The barcode read count distribution for the flowcell will be written to a
worksheet with title [date]_[flowcell id]. Additionally,  the read count per
project and sample can be summarized and written to a spreadsheet named
[project name]_sequencing results. The  spreadsheet will be tagged with the
project name and placed under the folder specified via the
'gdocs_projects_folder' configuration option.

## Getting v 0.3a working

Make sure to have the environment variable LC_ALL set to en_US.utf-8 by doing

    export LC_ALL=en_US.utf-8

## Getting v 0.3 a working

There is some required software which need be installed locally.

* snpEff 2.0.2

Version 2.0.3 is not compatible with GATK, make sure it is 2.0.2.
Also get the following data for snpEff 2.0.2

* GRCH37.63
* NCBIM37.73
* athalianaTair10

All for snpEff 2.0.2! It is all available on the snpEff download page.

Make sure to have the environment variable LC_ALL set to en_US.utf-8 by doing

    export LC_ALL=en_US.utf-8

Only changing LC_NUMERIC or LC_MONETARY does not work.

Also make sure to use

* tophat 1.3.0

or later.
