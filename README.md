# GenomeFastScreen [![license](https://img.shields.io/badge/license-MIT-brightgreen)](https://github.com/pegi3s/pss-genome-fs) [![dockerhub](https://img.shields.io/badge/hub-docker-blue)](https://hub.docker.com/r/pegi3s/pss-genome-fs) [![compihub](https://img.shields.io/badge/hub-compi-blue)](https://www.sing-group.org/compihub/explore/5e2eaacce1138700316488c1)
> **GenomeFastScreen** is a [Compi](https://www.sing-group.org/compi/) pipeline that automates the process of identifying genes that likely show PSS, and thus, should be studied in detail, starting from a set of FASTA files, one per genome, containing all coding sequences. To do this, it uses internally our previous **FastScreen** pipeline<sup>1</sup> (available [here](https://www.sing-group.org/compihub/explore/5d5bb64f6d9e31002f3ce30a)). GenomeFastScreen automatically removes problematic sequences such as those showing ambiguous positions and identifies orthologous gene sets. It is also possible to identify the orthologous genes in an external reference species, which is useful to compare results across different species, or conduct gene ontology enrichment analyses when there is no data for the species being analysed. A Docker image is available for this pipeline in [this Docker Hub repository](https://hub.docker.com/r/pegi3s/pss-genome-fs).

## GenomeFastScreen repositories

- [GitHub](https://github.com/pegi3s/pss-genome-fs)
- [DockerHub](https://hub.docker.com/r/pegi3s/pss-genome-fs)
- [CompiHub](https://www.sing-group.org/compihub/explore/5e2eaacce1138700316488c1)

# Using the GenomeFastScreen image in Linux

In order to use the GenomeFastScreen image, create first a directory in your local file system (`working_dir` in the example) with the following structure: 

```bash
working_dir/
├── input
│   ├── 1.fasta
│   ├── 2.fasta
│   ├── .
│   ├── .
│   ├── .
│   └── n.fasta
├── global
│   └── global-reference-file.fasta
└── pss-genome-fs.params
```

Where:
- The input FASTA files to be analized must be placed in the `working_dir/input` directory.
- The global reference FASTA file is located at `working_dir/global/global-reference-file.fasta`.
- The Compi parameters file is located at `working_dir/pss-genome-fs.params`.

Using such folder structure, the Compi parameters file must look like:

```
input_data_dir=input
reference_file=1.fasta
blast_type=tblastx
global_reference_file=global/global-reference-file.fasta
```

Where:
- `input_data_dir` is set to `input`, which is the name of the directory that contains the input FASTA files to be analyzed (it is possible to use a different name than `input` as long as it is properly set in the Compi parameters file).
- `reference_file` is set to `1.fasta`, which is the name input FASTA file to use as reference for finding the orthologous gene sets.
- `blast_type` is set to the type of BLAST to use for finding the orthologous gene sets (either `blastn` or `tblastx`).
- `global_reference_file` is set to the location (relative to `the working_dir` directory) of the global reference FASTA file. Note that this file is optional and it can be ommited. When provided, the task `orthologs-reference-species` and `get-orthologs-reference-specie-results` of the pipeline are executed using it in order to identify the orthologous genes in the short lists produced by FastScreen.

Once this structure and files are ready, you should run and adapt the following commands to run the entire pipeline. Here, you only need to set `WORKING_DIR` to the right path in your local file system and `COMPI_NUM_TASKS` to the maximum number of parallel tasks that can be run.

```bash
WORKING_DIR=/path/to/pss-genome-fs/working_dir/
COMPI_NUM_TASKS=8

docker run --rm -v /tmp:/tmp -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKING_DIR}:/working_dir --rm pegi3s/pss-genome-fs --logs /working_dir/logs --params /working_dir/pss-genome-fs.params --num-tasks ${COMPI_NUM_TASKS} -- --host_working_dir ${WORKING_DIR} --compi_num_tasks ${COMPI_NUM_TASKS}
```

The following section explains the main pipeline outputs as well as how to proceed with the analysis of the results.

# Pipeline outputs

The main pipeline outpus created in the `working_dir` are:
- `/orthologs`: this directory contains one FASTA file, one for each gene, with the sequences from the analysed genomes (problematic sequences such as those containing in-frame stop codons and ambiguous positions are excluded).
- `/fast-screen`: results generated by the FastScreen pipeline after analyzing all genes. The main output from this directory is the `short_list` file, which contains the names of the FASTA files that likely show evidence for PSS and that have been copied to the `/short_list` directory. Six other output files are generated in this directory:
  - `FUBAR_short_list`: a file containing the names of the files where evidence for positive selection has been found by FUBAR.
  - `to_be_reevaluated_by_codeML`: a file containing the names of the files that where re-evaluated by CodeML.
  - `codeML_random_list`: a file containing the names of the files from which a random sequence sample was taken because they were too large to be analysed by CodeML.
  - `codeML_short_list`: a file containing the names of the files where PSS were detected by CodeML model M2a.
  - `negative_list`: a file containing the names of the files where no evidence for positive selection was found by either FUBAR or CodeML.
  - `files_requiring_attention`: a file containing the names of the files that could not be processed without error (usually because they have in frame stop codons that were introduced during the nucleotide alignment step). These files are copied to the `/files_to_re_run` directory each time the `fast-screen` step is executed.
- `/short_list_dir`: contains the FASTA files that likely show evidence for PSS and, therefore, should be analysed in detail.
- `/files_to_re_run`: this directory contains the FASTA files listed in the `/fast-screen/files_requiring_attention` file after each execution of the `fast-screen` step. The following subsection explains how to re-analyze this files.
- `orthologous_gene_list`: a file containing the orthologous genes in the external reference species.
- `orthologous_codeML_short_list`: file containing the orthologous gene names listed in the `/fast-screen/codeML_short_list`.
- `orthologous_FUBAR_short_list`: file containing the orthologous gene names listed in the `/fast-screen/FUBAR_short_list`.
- `orthologous_gene_short_list`: file containing the orthologous gene names listed in the `/fast-screen/short_list`.

Note that the `orthologous_*` lists are only created when the global reference file is provided as parameter to the pipeline.

# Re-analyzing the files requiring attention

After running the entire pipeline, the `/files_to_re_run` directory contains the FASTA files listed in the `/fast-screen/files_requiring_attention` file. These files could not be analyzed by the FastScreen pipeline after each execution of the `fast-screen` step, usually because they have in frame stop codons that were introduced during the nucleotide alignment step. In this case, our [CheckCDS](https://www.sing-group.org/compihub/explore/5f588ccb407682001ad3a1d5#readme) Docker image can help in automatically producing valid CDS files.

To re-analyze them, you should first fix the problems and put the updated FASTA files in the same directory. Then, you should execute only the `fast-screen` step by running the following command:

```bash
docker run --rm -v /tmp:/tmp -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKING_DIR}:/working_dir --rm pegi3s/pss-genome-fs --logs /working_dir/logs --params /working_dir/pss-genome-fs.params --num-tasks ${COMPI_NUM_TASKS} --single-task fast-screen -- --host_working_dir ${WORKING_DIR} --compi_num_tasks ${COMPI_NUM_TASKS}
```

This command simply introduces the `--single-task fast-screen` parameter to ask Compi to only run this step. When this step finishes, the new results will be added to the existing ones in the `/fast-screen` and, like before, the `/files_to_re_run` directory will contain the FASTA files that could not be analzyed in this new run.

Once you are done analyzing the problematic files, you can re-run the final two steps of the pipeline to copy the FASTA files that likely show evidence for PSS and to re-generate the orthologous gene lists. To do this, you should execute the following two commands:

```bash
docker run --rm -v /tmp:/tmp -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKING_DIR}:/working_dir --rm pegi3s/pss-genome-fs --logs /working_dir/logs --params /working_dir/pss-genome-fs.params --num-tasks ${COMPI_NUM_TASKS} --single-task get-short-list-files -- --host_working_dir ${WORKING_DIR} --compi_num_tasks ${COMPI_NUM_TASKS}

docker run --rm -v /tmp:/tmp -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKING_DIR}:/working_dir --rm pegi3s/pss-genome-fs --logs /working_dir/logs --params /working_dir/pss-genome-fs.params --num-tasks ${COMPI_NUM_TASKS} --single-task get-orthologs-reference-species-results -- --host_working_dir ${WORKING_DIR} --compi_num_tasks ${COMPI_NUM_TASKS}
```

This commands uses `--single-task` parameter to ask Compi to run the `get-short-list-files` and the `get-orthologs-reference-species-results` steps.

# Test data

The sample data is available [here](https://github.com/pegi3s/pss-genome-fs/raw/master/resources/pss-genome-fs.zip). Download, uncompress it and move to the `pss-genome-fs` directory, where you will find:

- A directory called `working_dir`, that contains the structure described previously.
- A file called `run.sh`, that contains the following commands (where you should adapt the `WORKING_DIR` path) to test the pipeline:

```bash
WORKING_DIR=/path/to/pss-genome-fs/working_dir/
COMPI_NUM_TASKS=8

docker run --rm -v /tmp:/tmp -v /var/run/docker.sock:/var/run/docker.sock -v ${WORKING_DIR}:/working_dir --rm pegi3s/pss-genome-fs --logs /working_dir/logs --params /working_dir/pss-genome-fs.params --num-tasks ${COMPI_NUM_TASKS} -- --host_working_dir ${WORKING_DIR} --keep_temporary_dir --compi_num_tasks ${COMPI_NUM_TASKS}
```

## Running times

- ≈ 144 minutes - 50 parallel tasks - Ubuntu 18.04.2 LTS, 96 CPUs (AMD EPYC™ 7401 @ 2GHz), 1TB of RAM and SSD disk.
- ≈ 158 minutes - 16 parallel tasks - Ubuntu 18.04.3 LTS, 12 CPUs (AMD Ryzen 5 2600 @ 3.40GHz), 16GB of RAM and SSD disk.
- ≈ 760 minutes - 10 parallel tasks - Ubuntu 16.04.3 LTS, 8 CPUs (Intel(R) Core(TM) i5-5200U CPU @ 2.20GHz), 16GB of RAM and SSD disk.

# For Developers

## Building the Docker image

To build the Docker image, [`compi-dk`](https://www.sing-group.org/compi/#downloads) is required. Once you have it installed, simply run `compi-dk build` from the project directory to build the Docker image. The image will be created with the name specified in the `compi.project` file (i.e. `pegi3s/pss-genome-fs:latest`). This file also specifies the version of compi that goes into the Docker image.

# References

- H. López-Fernández; C. P. Vieira; P. Ferreira; P. Gouveia; F. Fdez-Riverola; M. Reboiro-Jato; J. Vieira (2021) **On the identification of clinically relevant bacterial amino acid changes at the whole genome level using Auto-PSS-Genome**. *Interdisciplinary Sciences: Computational Life Sciences*. Volume 13, pp. 334–343. [![DOI](https://img.shields.io/badge/DOI-10.1007%2Fs12539--021--00439--2-blue)](https://doi.org/10.1007/s12539-021-00439-2)
- H. López-Fernández; C.P. Vieira; F. Fdez-Riverola; M. Reboiro-Jato; J. Vieira (2020) **Inferences on Mycobacterium leprae host immune response escape and antibiotic resistance using genomic data and GenomeFastScreen**. *14th International Conference on Practical Applications of Computational Biology & Bioinformatics: PACBB 2020*. L'Aquila, Italy. 7 - October [![DOI](https://img.shields.io/badge/DOI-https%3A%2F%2Fdoi.org%2F10.1007%2F978--3--030--54568--0__5-green)](https://doi.org/10.1007/978-3-030-54568-0_5)
- H. López-Fernández; P. Duque; N. Vázquez; F. Fdez-Riverola; M. Reboiro-Jato; C.P. Vieira; J. Vieira (2019) **Inferring Positive Selection in Large Viral Datasets**. *13th International Conference on Practical Applications of Computational Biology & Bioinformatics: PACBB 2019*. Ávila, Spain. 26 - June [![DOI](https://img.shields.io/badge/DOI-10.1007%2F978--3--030--23873--5__8-green)](https://doi.org/10.1007/978-3-030-23873-5_8)
