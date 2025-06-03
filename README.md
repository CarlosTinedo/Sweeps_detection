# Sweeps_detection
This GitHub repository contains the methodology, steps, and scripts used to carry out sweep detection with PoolHMM. Different tools have been used, including:      
PoolHMM      
PoPoolation      
Samtools      
Conda environments with Python 2.7      
Packages with specific versions of SpaCy and NumPy.


#!/bin/bash
#$ -cwd
#$ -V

source /users-d2/c.m.tinedo/.bashrc

perl /users-d2/c.m.tinedo/sweeps/popoolation/Variance-sliding.pl \
  --input /users-d2/c.m.tinedo/sweeps/S20660000922_sm_nm_y_p_nonX.pileup \
  --output /users-d2/c.m.tinedo/sweeps/popoolation/variance_S20660000922_sm_nm_y_p_nonX.pileup \
  --snp-output snps_nonX_usados_S20660000922.txt \
  --measure pi \
  --pool-size 100 \
  --fastq-type illumina \
  --min-count 2 \
  --min-coverage 20 \
  --max-coverage 100 \
  --window-size 10000 \
  --step-size 1

