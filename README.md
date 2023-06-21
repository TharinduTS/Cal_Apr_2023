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
```bash
for i in ./temp_headers/*temp_head; do sed -r -i 's/\b(:1-217471166|:1-181034961|:1-153873357|:1-153961319|:1-164033575|:1-154486312|:1-133565930|:1-147241510|:1-91218944|:1-52432566|:1-17610)\b//g' ./temp_headers/XT7_WY_no_adapt__sorted.bam_rg.bam-temp_head ; done
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


