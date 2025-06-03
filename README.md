# Sweeps_detection
This GitHub repository contains the methodology, steps, and scripts used to carry out sweep detection with PoolHMM. Different tools have been used, including:      
PoolHMM      
PoPoolation      
Samtools      
Conda environments with Python 2.7      
Packages with specific versions of SpaCy and NumPy.

BAM to Pileup:
First of all, we convert the BAM files to pileup format using the Samtools tool. This step is necessary because both nucleotide diversity calculation and sweep detection require this input structure.
Specifically, we use the following command line:

samtools mpileup -f PATH/reference.FASTA PATH/BAMfile > PATH/name.pileup

We will need a Conda environment that includes the Samtools tool. Additionally, a reference FASTA file is also required in order to map and convert the BAM file into a Pileup file.

Missing calculation and removing:

