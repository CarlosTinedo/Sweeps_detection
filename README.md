# Sweeps_detection
This GitHub repository contains the methodology, steps, and scripts used to carry out sweep detection with PoolHMM. Different tools have been used, including:      
- PoolHMM      
- PoPoolation      
- Samtools      
- Conda environments with Python 2.7      
- Packages with specific versions of SpaCy and NumPy.

## BAM to Pileup:

First of all, we convert the BAM files to pileup format using the Samtools tool. This step is necessary because both nucleotide diversity calculation and sweep detection require this input structure.
Specifically, we use the following command line:

```
  samtools mpileup -f PATH/reference.FASTA PATH/BAMfile > PATH/name.pileup
```

We will need a Conda environment that includes the Samtools tool. Additionally, a reference FASTA file is also required in order to map and convert the BAM file into a Pileup file.

## Missing calculation and removing:

Once the Pileup files are created, it is necessary to check for missing data, as this could affect the calculations by being considered as positions without variants. To do this, we use a BASH script that searches for positions without counts and calculates the total percentage.

The script is called missing_calculator.sh and is as follows:

```
#!/bin/bash

# Se comprueba que se haya pasado correctamente el input Pileup
if [ $# -lt 1 ]; then
    echo "Uso: $0 archivo.pileup [archivo_salida]"
    exit 1
fi
# Se asigna las variables
archivo="$1"
salida="${2:-resultado_incompletas.txt}"

if [ ! -f "$archivo" ]; then
    echo "El archivo '$archivo' no existe."
    exit 1
fi

# Procesamiento con awk. Se cuenta el total de líneas y luego las líneas que no tienen las seis colmnas, incompletas por las faltas de counts
awk -v salida="$salida" '
BEGIN {
    total = 0
    incompletas = 0
}
{
    total++
    if (NF < 6) {
        incompletas++
    }
}
END {
    porcentaje = (incompletas / total) * 100
    printf("Total de filas: %d\n", total) > salida
    printf("Filas con menos de 6 columnas: %d\n", incompletas) >> salida
    printf("Porcentaje de filas incompletas: %.2f%%\n", porcentaje) >> salida
    print "Resultados guardados en " salida
}
' "$archivo"

```
With this script, we will obtain the percentage of missing data, if any. This first allows us to determine whether a subsequent filtering step will be necessary to avoid errors in the calculation of nucleotide variability. Additionally, it also provides an indication of the sequencing quality of the pools.

Next, we must remove all lines that either lack counts or have missing information. We do this using the BASH script killer.sh. This script simply removes lines with missing values in columns 4 to 6.

```
#!/bin/bash
#$ -cwd
#$ -V

source /users-d2/c.m.tinedo/.bashrc

# Nombre del archivo de entrada
input="PATH/name.pileup"

# Nombre del archivo de salida
output="PATH/filtrado_name.pileup"

# Filtrar liíneas que no contienen la información para los counts y calidad, es decir,tienen missing
awk '!( $4 == 0 && $5 == "." && $6 == "." )' "$input" > "$output"
```

Once we have removed all the missing positions, we now proceed with the filtering, in this case focusing on the chromosomes or DNA of interest.

## Remove mitochondrial DNA and the Y chromosome

We only use the four autosomal chromosomes and the X sex chromosome; all other genetic structures must be removed (this applies in the case of Drosophila melanogaster). Therefore, we must eliminate the Y sex chromosome, since all individuals are males, a feature stemming from the need to distinguish between different Drosophila species, and also the mitochondrial DNA, due to its high variability.

To perform this function, we use the BASH script called remove_mito.sh.

```
#!/bin/bash

# Asegurarse del input de entrada
if [ $# -lt 1 ]; then
    echo "Uso: $0 archivo.pileup [archivo_salida]"
    exit 1
fi

entrada="$1"
salida="${2:-sin_mito_y.pileup}"

if [ ! -f "$entrada" ]; then
    echo "El archivo '$entrada' no existe."
    exit 1
fi

# Eliminamos líneas que contienen 'Dsim_mitochondrion' o 'Dsim_Y', aquí lo pone para simulans, pero lo mismo para melano, eso si poniendo su nomenclatura
grep -Ev 'Dsim_mitochondrion|(^|[[:space:]])Dsim_Y([[:space:]]|$)' "$entrada" > "$salida"

# Opcional: contar líneas eliminadas
eliminadas=$(grep -Ec 'Dsim_mitochondrion|(^|[[:space:]])Dsim_Y([[:space:]]|$)' "$entrada")

```

## Distribution of chromosomes in two files

Once we have completed all this filtering, we can start processing the input and focus on calculating pi and searching for sweeps. However, before that, since the autosomal chromosomes and the X chromosome differ in variability and effective size, we must separate them—creating one file for the autosomes and another pileup containing only the X chromosome from the pool.

To carry out this part of the process, we use a BASH script called separador_pileup.sh. This script uses grep to search for the X chromosome in the file and export it to a new file, and with the -v option, we do the opposite: we export everything that is not the X chromosome.

Below, I present the script with comments.

```
#!/bin/bash
#$ -cwd
#$ -V

# Verificamos el archivo de entrada
if [ $# -ne 1 ]; then
    echo "Uso: $0 archivo.pileup"
    exit 1
fi

PILEUP_FILE="$1"
BASENAME=$(basename "$PILEUP_FILE" .pileup)

# Definimos los archivos de salida
X_LINES="${BASENAME}_X.pileup"
OTHER_LINES="${BASENAME}_nonX.pileup"

# Creamos los nuevos pileup con el comando grep
grep '^Dsim_X' "$PILEUP_FILE" > "$X_LINES"
grep -v '^Dsim_X' "$PILEUP_FILE" > "$OTHER_LINES"
```

This script, in our case, requires a launcher script for the Hércules node, where the input or original pileup file to be used is defined and where execution permissions are granted.

The name of this script is distribuidora.sh, is the next one.

```
#!/bin/bash
#$ -cwd
#$ -V

chmod +x separador_pileup.sh

SCRIPT_PATH/separador_pileup.sh ORIGINAL_PILEUP_PATH/original.pileup

```

## Nucleotide diversity calculation

To calculate nucleotide diversity, we used the genomic software Popoolation.
This software computes nucleotide diversity, that is, theta (θ) per position. To do this, it requires a pileup file from the pool under study. In addition to providing the probability of divergence within a population for each position in the sequence, it also gives an estimate of the effective population size (Ne), since theta = 4 * Ne * μ.

The software is based on a Perl script called Variance-sliding.pl, which performs the calculations using sliding windows defined by the user through parameters.

The required parameters are:

- The statistic or measure we want to compute, in our case π (pi)
- The pool size, referring to the number of haplotypes (i.e., number of chromosomes)
- The sequencing method
- The minimum count required for a position to be considered a SNP
- The minimum coverage for a position to be taken into account
- The maximum coverage
- The window size, in our case 10,000 positions
- The step size between windows, which we set to 1 so that there are no gaps but also no overlaps, thus covering the entire genome

To run the genomic software Popoolation, we use the script called popoolation.sh, where we call its Perl script, define the input, output, and all the previously mentioned parameters.

```
#!/bin/bash
#$ -cwd
#$ -V

source /users-d2/c.m.tinedo/.bashrc

perl /users-d2/c.m.tinedo/sweeps/popoolation/Variance-sliding.pl \
  --input /users-d2/c.m.tinedo/sweeps/popEXAMPLE_sm_nm_y_p_nonX.pileup \
  --output /users-d2/c.m.tinedo/sweeps/popoolation/variance_popEXAMPLE_sm_nm_y_p_nonX.pileup \
  --snp-output snps_nonX_usados_popEXAMPLE.txt \
  --measure pi \
  --pool-size 100 \
  --fastq-type illumina \
  --min-count 2 \
  --min-coverage 20 \
  --max-coverage 100 \
  --window-size 10000 \
  --step-size 1
```

Once the results from Popoolation are obtained, we do not have the nucleotide diversity per pool, but rather per position. To obtain the global value, we need to filter and calculate the average. First, we filter out the positions with no data using the BASH script colador.sh.

```
#!/bin/bash

input="variance_popEXAMPLE_sm_nm_y_p_nonX.pileup"

output="filtrado_variance_popEXAMPLE_sm_nm_y_p_nonX.pileup"

awk '$4 != "0.000"' "$input" > "$output"

```

Once we obtain the filtered outputs, we calculate the average using the AWK command directly in the Linux terminal.

```
awk '{sum += $4} END {print sum/NR}' filtrado_variance_popEXAMPLE_sm_nm_y_p_nonX.txt
```

## Sweep detection with Pool-HMM

Once we have the nucleotide variability, we can use it as a flag in the Pool-HMM software to detect sweeps.
Pool-HMM is a software that detects hard sweeps from pools. It is built in Python and requires the pool’s pileup file, the program’s Python script, the number of chromosomes, theta, k (the probability of a model change), the minimum coverage, and the number of CPUs to parallelize the process.

Pool-HMM scans the genome in search of regions lacking variability, which may indicate a possible sweep. It classifies positions into three models:

1 -> Neutral model: positions not under the effect of selection.
2 -> Purifying selection model: positions where deleterious variants are removed by natural selection.
3 -> Positive selection model: positions under directional selection, where a new advantageous mutation rapidly increases in frequency. This is the model of interest to us.

One limitation of Pool-HMM is that it does not distinguish positions by chromosome, so we must differentiate them manually. Otherwise, it merges positions from different chromosomes and produces sweeps that do not actually exist.

Below, I explain the parameters used in Pool-HMM:

- K -> the probability of a model change from one position to another. This parameter has been defined based on the literature, although other authors have obtained it through simulations.
- Theta -> an estimate of nucleotide variability, which is related to Me. It has been calculated using the Popoolation software.
- n -> estimated based on the number of individuals in the pool and whether the chromosome is autosomal or sex-linked.
- c -> the minimum coverage a position must have to be considered.
- P -> the number of CPUs to be used for process parallelization.

Attached is the BASH script used to run the genomic software, called prove_tfg.sh:

```
#!/bin/bash
#$ -cwd
#$ -V

pop="popEXAMPLE"


source /users-d2/c.m.tinedo/.bashrc
conda activate sweeps_tini

for i in 2L 2R 3L 3R

do

  grep "$i" /users-d2/c.m.tinedo/sweeps/parametres/${pop}_filtrado_tab.pileup > /users-d2/c.m.tinedo/sweeps/parametres/${pop}_${i}.pileup
  python /users-d2/c.m.tinedo/sweeps/poolhmm/PoolHMM-master/pool-hmm.py \
        --prefix /users-d2/c.m.tinedo/sweeps/parametres/${pop}_${i} \
        -n 100 \
        --pred \
        -k 0.0000000001 \
        --theta 0.005 \
        -c 15 \
        -r 20 \
        -P 10
done`

```

Two tests were performed using Pool-HMM. One with k = 0.00001 and another with k = 0.0000000001. In other words, we conducted one more liberal test and another more conservative one to eliminate false positives, although a manual check is later carried out to see which region of the chromosome each one belongs to.
