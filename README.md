# MHConstructor
# Project Background
Pipeline description: [MHConstructor: a high-throughput, haplotype-informed solution to the MHC assembly challenge](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-024-03412-6)

# What is MHC, and why does it matter to neurodegenerative disease?
The major histocompatibility complex (MHC), the most polymorphic region of the human genome, encodes human leukocyte antigen (HLA) genes that play a key role in innate immunity and have been strongly associated with neurodegenerative diseases. Microglia, the brain’s resident immune cells, are regulated by MHC signaling, and when overactivated, contribute to neuroinflammation and disease progression. However, the specific mechanisms and causal variants mediating this response remain poorly understood. 

# Why do we need MHConstructor?
Due to its complex structure, marked by high heterozygosity and structural variation, the MHC region is challenging to sequence with traditional reference-based tools. To address this, our collaborators in the Hollenbach lab at UCSF have developed [MHConstructor](https://github.com/Hollenbach-lab/MHConstructor), a novel tool that enables reference-free, de novo assembly of the MHC locus, allowing detection of variants that are not represented in current reference genomes. 

# Install Necessary Tools
Download the MHConstructor repository
```
git clone https://github.com/Hollenbach-lab/MHConstructor.git
cd ./MHConstructor
```
Ensure you have Singularity downloaded or see other options for download on the official [GitHub](https://github.com/Hollenbach-lab/MHConstructor) page to build the image.
```
singularity pull mhconstructor.sif library://rsuseno/rsuseno/mhconstructor:latest
```

Download [C4Investigator](https://github.com/Hollenbach-lab/C4Investigator)(Marin et al., 2024)
```
git clone https://github.com/Hollenbach-lab/C4Investigator.git
cd C4Investigator
```
Build Singularity Container 
```
singularity build --fakeroot c4investigator.sif c4investigator.def
```
OR if you don't want to use --fakeroot, try remote login
```
singularity remote login
singularity build --remote c4investigator.sif c4investigator.def
```
Download samtools if you don't already have it
```
# Set installation directory
INSTALL_DIR=$HOME/.local
mkdir -p $INSTALL_DIR

# Download and install HTSlib
cd ~
wget https://github.com/samtools/htslib/releases/download/1.20/htslib-1.20.tar.bz2
tar -xvjf htslib-1.20.tar.bz2
cd htslib-1.20
./configure --prefix=$INSTALL_DIR
make
make install

# Download and install Samtools
cd ~
wget https://github.com/samtools/samtools/releases/download/1.20/samtools-1.20.tar.bz2
tar -xvjf samtools-1.20.tar.bz2
cd samtools-1.20
./configure --prefix=$INSTALL_DIR --with-htslib=$INSTALL_DIR
make
make install

# Update PATH and LD_LIBRARY_PATH
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$HOME/.local/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc

# Verify installation
samtools --version
```
Download conda if you don't already have in your environment.
```
#Install miniconda (without sudo)
# Download Miniconda installer
cd ~
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh
# Run the installer
bash miniconda.sh -b -p $HOME/miniconda
# Initialize conda in your shell
$HOME/miniconda/bin/conda init bash
source ~/.bashrc
# Verify installation
conda --version
```
# Usage Guide 
Example scripts can be found in 
```/yokoyama/seqdata/projects/MHConstructor```
Update the ```control.txt``` file in your MHConstructor folder with your updates folder paths
Create a file like ```file_ids_MAC4kWGS.txt``` with your cram ids in a line separated list like this:
```
SAMPLEID_1
SAMPLEID_2
```

First run ```1_run_extractFastq.sh``` to extract fastqs from .cram files
```
#!/bin/bash
# run this script first to extract fastqs from .cram files
# nohup bash 1_run_extractFastq.sh > MAC4kWGS_run2.log 2>&1 &

set -euo pipefail

# cd into your folder with your MHConstructor  scripts
cd /yokoyama/seqdata/projects/MHConstructor/

# Directories -- update with your file paths 
CRAM_DIR="/yokoyama/MAC4kWGS/crams/MAC4k_subset_9.12.25"
REF_FASTA="$HOME/hg38.fa"
MHCONSTRUCTOR_DIR="$HOME/MHConstructor"
OUT_DIR="$HOME/MAC4kWGS/cramToFastqOut"
ID_FILE="/yokoyama/seqdata/projects/MHConstructor/file_ids_MAC4kWGS.txt"
SAMTOOLS_BIN="$HOME/samtools-1.20/samtools"

# Where YOU want indexes stored:
INDEX_DIR="$HOME/MAC4kWGS/cram_indexes"
mkdir -p "$INDEX_DIR"

echo "[INFO] Indexing CRAMs into: $INDEX_DIR"

# Index each CRAM to your own folder
while read -r ID; do
    CRAM="${CRAM_DIR}/${ID}.cram"
    CRAI="${INDEX_DIR}/${ID}.crai"

    if [[ -f "$CRAM" ]]; then
        echo "[INFO] Indexing $ID"
        $SAMTOOLS_BIN index -o "$CRAI" "$CRAM"
    else
        echo "[WARNING] CRAM not found for ID $ID"
    fi
done < "$ID_FILE"

echo "[INFO] Finished indexing all samples."

# Now run the extraction pipeline (no changes here)
nohup bash extractFastqFromCram_toDir_w_logging.sh \
  "$CRAM_DIR" \
  "$REF_FASTA" \
  "$MHCONSTRUCTOR_DIR" \
  "$OUT_DIR" \
  "$ID_FILE" \
  "$SAMTOOLS_BIN" \
  > run_extract_MAC4kWGS_test.log &
```
This script will call ```extractFastqFromCram_toDir_w_logging.sh```
```
#! /bin/bash

## Script to iterate through all *.cram files in a provided directory ($1)
## and extract region associated with the MHC reference coordinates
## for hg38 reference genome
## wadekj 
## 05/15/23
## revised: 5/2/25

## User input args:
# $1 - name of directory containing cram files. If current directory, use "./"
	## Otherwise, like this: "/home/user/exampleDir/"
# $2 - reference genome. If file on local server, list full directory and file name: "/home/user/exampleDir/refGenome.fasta" Otherwise, if on remote server, input full ftp path
# $3 - path to MHConstructor directory
# $4 - assembly directory
# $5 -  text file containing new line delim list of cram file ids

set -euo pipefail

progSamtools=${6}
#progSamtools=${3}/MHConstructor/tools/samtools-1.9/samtools

ids="$(cat ${5})"

#added logging for error tracing
for i in ${ids};
do 
        cramfile="$(find ${1} -name ${i}*.cram)"
        if [[ -z "${cramfile}" ]]; then
            echo "[ERROR] No CRAM file found for ID ${i}"
            continue
        fi

        echo ${cramfile}
        fname=$(basename "${cramfile}" .cram)
        echo ${fname}
        dirName=${4}
        mkdir -p ${dirName}

        echo "[INFO] Indexing CRAM"
        # ${progSamtools} index ${cramfile}
        ${progSamtools} index -o ${dirName}/${fname}.crai ${cramfile}

	    ## Get MHC.bam from .cram
        echo "[INFO] Converting CRAM to BAM"
        ${progSamtools} view -T ${2} -b ${cramfile} -o ${dirName}/${fname}.bam

        echo "[INFO] Sorting BAM"
        ${progSamtools} sort ${dirName}/${fname}.bam -o ${dirName}/${fname}.sort.bam
        echo 'done sort'

        echo "[INFO] Indexing sorted BAM"
        ${progSamtools} index ${dirName}/${fname}.sort.bam
        echo 'done index'

        echo "[INFO] Extracting MHC paired reads"
        ${progSamtools} view -f 1 -h -T ${2} -b ${dirName}/${fname}.sort.bam chr6:28525013-33457522 \
            | ${progSamtools} sort -n -o ${dirName}/${fname}_mhc_paired.sorted.bam

        echo "[INFO] Extracting unmapped reads"
        ${progSamtools} view -h -T ${2} -b ${dirName}/${fname}.sort.bam -o ${dirName}/${fname}_unmap.bam '*' \
            && ${progSamtools} sort -n -o ${dirName}/${fname}_unmap.sorted.bam ${dirName}/${fname}_unmap.bam

        echo "[INFO] Moving .crai index files"
        mv ./*.crai ${dirName} || true  # ignore error if none found

        echo "[INFO] Creating FASTQ for MHC paired reads"
        ${progSamtools} fastq -n ${dirName}/${fname}_mhc_paired.sorted.bam \
            -1 ${dirName}/${fname}_mhc_paired_R1.fastq \
            -2 ${dirName}/${fname}_mhc_paired_R2.fastq \
            -0 /dev/null -s /dev/null

        echo "[INFO] Creating FASTQ for unmapped reads"
        if [[ -f ${dirName}/${fname}_unmap.sorted.bam ]]; then
            ${progSamtools} fastq -n ${dirName}/${fname}_unmap.sorted.bam \
                -1 ${dirName}/${fname}_mhc_unmap_R1.fastq \
                -2 ${dirName}/${fname}_mhc_unmap_R2.fastq \
                -0 /dev/null -s /dev/null
        else
            echo "[WARNING] Unmapped BAM not found for ${fname}"
            touch ${dirName}/${fname}_mhc_unmap_R1.fastq ${dirName}/${fname}_mhc_unmap_R2.fastq
        fi
        ### Merge the mapped and unmapped reads together
        echo "[INFO] Merging FASTQs"
        cat ${dirName}/${fname}_mhc_paired_R1.fastq ${dirName}/${fname}_mhc_unmap_R1.fastq > ${dirName}/${fname}_mhc_all_R1.fastq
        cat ${dirName}/${fname}_mhc_paired_R2.fastq ${dirName}/${fname}_mhc_unmap_R2.fastq > ${dirName}/${fname}_mhc_all_R2.fastq

        echo "[SUCCESS] Finished ${fname}"

done;
```
Next run ```2_run_C4investigator.sh``` to run C4 investigator
```
#!/bin/bash
# make sure to update lines 291/ 302 in C4Investigator_run.R with intended file output name to not overwrite prior results
# update with your log file path
LOGFILE="/yokoyama/seqdata/projects/MHConstructorC4Investigator_run_$(date +%Y%m%d_%H%M%S).log"
echo "[INFO] Starting C4Investigator run at $(date)" | tee -a "$LOGFILE"
echo "[INFO] Listing FASTQ files matching pattern 'mhc_paired_R':" | tee -a "$LOGFILE"
# update with your fastq file output
ls ~/MAC4kWGS/cramToFastqOut/*mhc_paired_R*.fastq* 2>&1 | tee -a "$LOGFILE"

# update with the directory location for your cramToFastqOut and your MHConstructor directory
echo "[INFO] Running C4Investigator..." | tee -a "$LOGFILE"
singularity exec --bind ~/MAC4kWGS/cramToFastqOut,~/MHConstructor/genotypes c4investigator.sif \
  Rscript C4Investigator_run.R \
  -f ~/MAC4kWGS/cramToFastqOut \
  -r ~/MHConstructor/genotypes \
  --fastqPattern fastq \
  -t 8 2>&1 | tee -a "$LOGFILE"

echo "[INFO] C4Investigator run finished at $(date)" | tee -a "$LOGFILE"
```

Next run ```3_run_HLA-DRB1_haplotypes.sh``` to run T1K to gather HLA-DRB1 haplotypes
```
cd ~/MHConstructor/genotypes/
singularity exec ../container/mhconstructor.sif /bin/bash genotypeDRB.sh /yokoyama/seqdata/projects/MHConstructorfile_ids_MAC4kWGS_test.txt
cd ..
```
Next run ```4_run_MHConstructor_MAC4k.sh``` to run MHConstructor
```
#!/bin/bash

cd ~/MHConstructor || exit 1

LOGFILE="~/MHConstructor_scripts/MHConstructor_scripts_final/pipeline_run_$(date +%Y%m%d_%H%M%S).log"

echo "[INFO] Starting MHConstructor pipeline" | tee "${LOGFILE}"
echo "[INFO] Timestamp: $(date)" | tee -a "${LOGFILE}"

singularity exec container/mhconstructor.sif /bin/bash MHCgenerate.sh ~/MHConstructor_scripts/file_ids_complete.txt \
  | tee -a "${LOGFILE}"

echo "[INFO] Pipeline finished at $(date)" | tee -a "${LOGFILE}"
```
