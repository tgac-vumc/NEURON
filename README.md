# 🧠 NEURON

## 🌟 Highlights

- Nanopore methylation-based brain tumor diagnosis (Sturgeon)
- Copy Number Profiles (QDNAseq)
- Estimated tumor purity (ACE)
- _MGMT_ promoter methylation status (Rapid-CNS2)


## ℹ️ Overview

NEURON (NEURo-Oncology Nanopore) was developed for the molecular diagnostics department of Amsterdam UMC and can be used for automated analysis of nanopore-sequencing data, used for diagnostics of brain tumors
It is tailored for the SLURM-controlled HPC environment 'Helios' of Amsterdam UMC, and containerized as an Apptainer/Singularity image, built from the definition files in singularity/ (on top of the `rocker/r-ver` base image from Docker Hub). Starting from basecalled and aligned nanopore bam files, the pipeline includes the Sturgeon v1 and Sturgeon v2 CNS tumor classifiers (Vermeulen et al. 2023 and Vermeulen et al. 2026), _MGMT_ promoter methylation status (Patel et al. 2022), CNV profiling using QDNAseq, and tumor cell fraction estimation inferred from the copy number data using ACE (Poell et al. 2019), and compiles it into 1 PDF report.

In addition, this pipeline also offers a reconstructed timeline analysis based on the ONT's sequencing summary file. You can predefine the time bins. Each bin gives you Sturgeon V1 and V2 outcomes as well as a bin-corresponding CNV profile. This is visualized in line graphs over time, and again compiled into 1 PDF report.

<img width="3466" height="2100" alt="NeuroTide pipeline" src="https://github.com/user-attachments/assets/227527e3-0b24-46fc-b521-eb6dfc5ac934" />

### ✍️ Authors

Bernard Stikker, PhD - Department of Pathology - Amsterdam UMC


## 🚀 Usage

First, we define our sample names. The pipeline will look in the input/ folder for bam files and it's relative location should be defined in the config/samples.yaml file. Input bam files should be merged per sample and aligned to the T2T-CHM13v2.0 reference genome. For example:

```bash
samples:
  test_data:
    bam: "NEURON/input/test_data/test_sample.bam"
    sequencing_summary: "NEURON/input/test_data/sequencing_summary.txt"
    barcode_label: "test_data"
```

You can configure your SLURM settings in the profile/config.yaml file. In this way, you can run the pipeline as follows:

```bash
snakemake --profile profile 
```
In addition, the pipeline also offers an (optional) timecourse analysis next to the regular pipeline. In your config/samples.yaml, add the location of ONT's sequencing summary file as well as the name you gave your samples prior to the sequencing run (barcode_label). This label should match exactly to the 'alias' column in the sequencing summary. Rename the sequencing summary file exactly to 'sequencing summary.txt'. Automated detection is in development.

This feature can be run as follows:

```bash
snakemake --profile profile all_timecourse
```

### Running locally (without SLURM)
NEURON does not need an HPC cluster. Every rule runs in the same container, so you can run the whole pipeline on a single Linux workstation or server with Snakemake and Apptainer/Singularity. You don't need the SLURM plugin or `profile/config.yaml`. Follow the Installation steps below (Snakemake, Apptainer, container build, models). Then run from the repository root:

```bash
snakemake --cores 8 --sdm apptainer --resources mem_mb=30000
```

- `--cores`: the number of CPU cores NEURON may use. Rules that ask for more threads in `config/samples.yaml` (e.g. Sturgeon: 10) are scaled down to this number.
- `--sdm apptainer`: runs each rule inside the container set by `container:` in `config/samples.yaml` (same as `use-singularity: true` in the SLURM profile).
- `--resources mem_mb=...`: the memory (MB) NEURON may use in total. Set this a bit below your machine's RAM. Snakemake then only runs jobs side by side if they fit within this limit.

#### Memory and disk use
The memory requests in the Snakefile (32–128 GB for the Sturgeon step, depending on BAM size) are set generously so that jobs are not killed on the HPC. They are not a minimum. Locally, Snakemake lowers any request that is larger than `--resources mem_mb` to that limit and runs the job anyway.

How much memory a run really needs depends on sequencing depth, not on the classifiers. Sturgeon v1 and v2 themselves are light. Most memory goes into the Sturgeon v1 preprocessing step (`sturgeon inputtobed`) and the _MGMT_ step. Both read the full output of `modkit extract`, which has one line per CpG call per read. A small run, such as an intraoperative sample, therefore needs much less memory than a deeply sequenced sample. The same file also takes up disk space in `results/<sample>/` during the run, so make sure there is enough free disk space. It is removed when the run finishes.

The timecourse analysis runs the same way:

```bash
snakemake all_timecourse --cores 8 --sdm apptainer --resources mem_mb=30000
```
Put the target (`all_timecourse`) before `--resources`. Otherwise Snakemake reads it as another resource.

Apptainer mounts your home directory, the current directory and `/tmp` into the container by default. If your BAM files or the repository are on another disk, mount that location as well, e.g.:

```bash
snakemake --cores 8 --sdm apptainer --resources mem_mb=30000 --apptainer-args "--bind /data:/data"
```

Check what will run first with a dry run by adding `-n`. To test the installation, use the test set in `input/test_data/` and set the sample paths in `config/samples.yaml` to that folder (see [Test the pipeline](#test-the-pipeline)).

Apptainer only runs on Linux. On macOS or Windows, use a Linux virtual machine or WSL2.

### Final report
Each sample gets `results/<sample>/final_report.pdf` with:
- Key results: top Sturgeon v1 and v2 class, _MGMT_ promoter status and tumour cell fraction
- Classifier scores: the top 5 classes of Sturgeon v1 and v2 with their raw scores and the number of probes/CpG sites used
- Classifier plots, copy-number profile with ACE and QDNAseq QC, and _MGMT_ details
- Software and reference data: versions of the pipeline (with git commit), container, classifiers, tools and R packages, and MD5 checksums of the model and reference files

The same version information is saved as `results/<sample>/software_versions.tsv`.

## ⬇️ Installation

### If you haven't installed snakemake yet, this pipeline has been validated using version 9.8.1.
```bash
conda install -c conda-forge -c bioconda snakemake=9.8.1
```
### As Helios uses SLURM as task manager, install the SLURM plugin
This is only needed on a SLURM cluster, not for [running locally](#running-locally-without-slurm).
```bash
pip install snakemake-executor-plugin-slurm
```
### Install Apptainer (Singularity)
NEURON runs every step in an Apptainer container, so Apptainer must be installed. Apptainer is the open-source successor of Singularity; either works. On Helios it is available as a module (`singularity` 1.4.1-1.el9). On an HPC cluster, check whether it is available as a module before installing it yourself (e.g. `module avail apptainer`). Check an existing install with:
```bash
apptainer --version
```

On your own Linux machine, install it with the package manager (needs sudo). See the [Apptainer installation guide](https://apptainer.org/docs/admin/main/installation.html) for other systems.

Ubuntu:
```bash
sudo apt update && sudo apt install -y software-properties-common
sudo add-apt-repository -y ppa:apptainer/ppa
sudo apt update && sudo apt install -y apptainer
```

Debian: download the `.deb` package of the [latest release](https://github.com/apptainer/apptainer/releases) and install it:
```bash
sudo apt install -y ./apptainer_<version>_amd64.deb
```

RHEL, Rocky, AlmaLinux (via EPEL) and Fedora:
```bash
sudo dnf install -y epel-release   # not needed on Fedora
sudo dnf install -y apptainer
```

Without sudo, you can install it in a folder of your choice (needs `curl`, `rpm2cpio` and `cpio`), then add `<folder>/bin` to your `PATH`:
```bash
curl -s https://raw.githubusercontent.com/apptainer/apptainer/main/tools/install-unprivileged.sh | bash -s - <folder>
```

Apptainer only runs on Linux. On macOS or Windows, use a Linux virtual machine or WSL2.

### Install container from .def file
Run the build from the repository root (the NEURON/ folder), not from inside singularity/: the definition file copies `singularity/sturgeon.yml` using a path relative to the folder you build from.
```bash
cd NEURON
apptainer build --fakeroot singularity/cns_AUMC_pipeline_v2.3.sif singularity/cns_AUMC_pipeline_v2.3.def
```
This writes the container to `singularity/cns_AUMC_pipeline_v2.3.sif`, where config/samples.yaml expects it.

The container installs Sturgeon V2 from https://github.com/deRidderLab/sturgeon-v2, pinned to a fixed commit, so the build needs internet access to GitHub. The previous definition (v2.2, with a local copy of Sturgeon V2) is kept in singularity/ for reference.
### Reference genome
The pipeline does not need a reference FASTA, but input BAM files must be aligned to T2T-CHM13v2.0 (https://s3-us-west-2.amazonaws.com/human-pangenomics/T2T/CHM13/assemblies/analysis_set/chm13v2.0.fa.gz), because the probe locations, QDNAseq bins and gene annotations are all in T2T-CHM13v2.0 coordinates.

### QDNAseq bins
The CNV analysis uses `reference/bins/T2T.500kbp.SR50.rda` (set by `qdnaseq_bins` in config/samples.yaml). The other bin sizes in that folder are kept for future development and are not used by the pipeline. See reference/bins/README.md before switching bin size.

### Downloading the classifiers (software is installed through the .def file, but models need to be downloaded manually)
Sturgeon V1: https://github.com/UMCUGenetics/sturgeon
Sturgeon V2: https://github.com/deRidderLab/sturgeon-v2 (model file `cns-v2.5.zip`: https://zenodo.org/records/22899108)

Move the model zip files to NEURON/reference/models/ (expected names: `general.zip` for V1, `cns-v2.5.zip` for V2). The models are not part of the container; the pipeline reads them from this folder. To use a different location or file name, change `sturgeon_model` / `sturgeon_v2_model` under `references` in config/samples.yaml.

### Test the pipeline
A small testset has been added to the input folder to test correct installation of the workflow. It does not yield biologically meaningful outcomes, but is only to be used to test the workflow.

## 💭 Feedback and Contributing

Add a link to the Discussions tab in your repo and invite users to open issues for bugs/feature requests.
Please do not hesitate to report bugs or to discuss issues!: https://github.com/tgac-vumc/NEURON/issues
