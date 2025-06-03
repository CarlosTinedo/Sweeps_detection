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
