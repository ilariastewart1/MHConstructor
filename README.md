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
# Usage Guide 
