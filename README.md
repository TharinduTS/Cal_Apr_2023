# Cal_Apr_2023

First do something like this:

samtools view -H old_bamfile.bam > temp_header.txt

then edit temp_header.txt to change the chromosome names to the ones you want

then use samtools reheader to replace the header of the bam file:
samtools reheader temp_header.txt old_bamfile.bam > new_bamfile.bam

as detailed here:
http://www.htslib.org/doc/samtools-reheader.html

After this I suggest focusing the Fst genomic windows on only the first 50 Mb of Chr7. With this region only you can try different sizes of genomic windows...

# renaming headers as chr names gives issues
make a directory for temp headers
```bash
mkdir temp_headers
```
seperate temp headers for all files
```bash
for i in ./*.bam; do samtools view -H $i > ./temp_headers/$i-temp_head ; done
```
edit the headers in temp_headers folder (this removes unessential numbers starting with ":"
run this inside temp head folder
```bash
for i in *temp_head; do sed -r -i 's/\b(:1-217471166|:1-181034961|:1-153873357|:1-153961319|:1-164033575|:1-154486312|:1-133565930|:1-147241510|:1-91218944|:1-52432566|:1-17610)\b//g' $i ; done
```
then rehead the files in the directory with original files

first create a directory
```bash
mkdir reheaded_files
```
then
```
for i in ./*.bam; do samtools reheader temp_headers/$i-temp_head $i > reheaded_files/$i-reheaded ; done
```
index reheaded files (inside reheaded_files directory

```
for i in *reheaded; do samtools index $i ; done
```

extract first 50mb of chr 7

make a seperate directory inside reheaded_files directory

```
mkdir Chr7_50mb
```
then
```bash
for i in *reheaded; do samtools view -b $i "Chr7:1-50000000" > Chr7_50mb/$i-Chr7_50mb ;done
```
Rename all the files to include .bam
```bash
for i in *reheaded; do mv $i $i.bam ; done
```
Create population files

pop1
```
F_Ghana_WZ_BJE4687_combined__sorted.bam_rg.bam-reheaded.bam
F_IvoryCoast_xen228_combined__sorted.bam_rg.bam-reheaded.bam
F_Nigeria_EUA0331_combined__sorted.bam_rg.bam-reheaded.bam
F_Nigeria_EUA0333_combined__sorted.bam_rg.bam-reheaded.bam
F_SierraLeone_AMNH17272_combined__sorted.bam_rg.bam-reheaded.bam
F_SierraLeone_AMNH17274_combined__sorted.bam_rg.bam-reheaded.bam
ROM19161__sorted.bam_rg.bam-reheaded.bam
XT10_WZ_no_adapt._sorted.bam_rg.bam-reheaded.bam
XT11_WW_trim_no_adapt_scafconcat_sorted.bam_rg.bam-reheaded.bam
```
pop2
```
M_Ghana_WY_BJE4362_combined__sorted.bam_rg.bam-reheaded.bam
M_Ghana_ZY_BJE4360_combined__sorted.bam_rg.bam-reheaded.bam
M_Nigeria_EUA0334_combined__sorted.bam_rg.bam-reheaded.bam
M_Nigeria_EUA0335_combined__sorted.bam_rg.bam-reheaded.bam
M_SierraLeone_AMNH17271_combined__sorted.bam_rg.bam-reheaded.bam
M_SierraLeone_AMNH17273_combined__sorted.bam_rg.bam-reheaded.bam
XT1_ZY_no_adapt._sorted.bam_rg.bam-reheaded.bam
XT7_WY_no_adapt__sorted.bam_rg.bam-reheaded.bam
```
cal Fst script

```
#!/bin/sh
#SBATCH --job-name=fst
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --time=168:00:00
#SBATCH --mem=512gb
#SBATCH --output=abba.%J.out
#SBATCH --error=abba.%J.err
#SBATCH --account=def-ben

#SBATCH --mail-user=premacht@mcmaster.ca
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --mail-type=REQUEUE
#SBATCH --mail-type=ALL

# Load modules
module load nixpkgs/16.09  gcc/7.3.0
module load angsd
module load gsl/2.5
module load htslib

# Define directory containing BAM files
bamdir=./

# Define output directory for Fst results
outdir=./FST_outs

# Define reference genome
refgenome=../../../2020_XT_v10_refgenome/XENTR_10.0_genome_scafconcat.fasta

# Loop over all BAM files in the directory
for bamfile in ${bamdir}/*.bam; do
    # Extract the sample name from the file name
    sample=$(basename ${bamfile} .bam)


#this is with 2pops
#first calculate per pop saf for each populatoin
angsd -b pop1  -anc ${refgenome} -out ${outdir}/${sample}_pop1 -dosaf 1 -gl 1 
angsd -b pop2  -anc ${refgenome} -out ${outdir}/${sample}_pop2 -dosaf 1 -gl 1 

#calculate the 2dsfs prior
realSFS ${outdir}/${sample}_pop1.saf.idx ${outdir}/${sample}_pop2.saf.idx >pop1.pop2.ml

#prepare the fst for easy window analysis etc
realSFS fst index ${outdir}/${sample}_pop1.saf.idx ${outdir}/${sample}_pop2.saf.idx -sfs pop1.pop2.ml -fstout here

#get the global estimate
realSFS fst stats here.fst.idx
```

copy pop1 pop2 and cal_Fst.sh to the reheaded folder and run the script

## Admixture

#prepare files for admixture and PCA with this script

```bash
#!/bin/sh
#SBATCH --job-name=fst
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --time=48:00:00
#SBATCH --mem=512gb
#SBATCH --output=abba.%J.out
#SBATCH --error=abba.%J.err
#SBATCH --account=def-ben

#SBATCH --mail-user=premacht@mcmaster.ca
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --mail-type=REQUEUE
#SBATCH --mail-type=ALL

# Load modules
module load nixpkgs/16.09  gcc/7.3.0
module load angsd
module load gsl/2.5
module load htslib

angsd -GL 1 -out genolike -nThreads 10 -doGlf 2 -doMajorMinor 1 -SNP_pval 1e-6 -doMaf 1  -bam bam.filelist
```

## PCA
you can use the prepared files from the above file as inputs here

Install PCAngsd

follow the guide in 

http://www.popgen.dk/software/index.php/PCAngsd

you may need the help in

https://github.com/TharinduTS/useful_commands/blob/master/README.md#python
