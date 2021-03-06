# Handling_AmpliSeq_Transciptome_Data

Basic linux commands for handling Ampliseq_transcriptome data

### Create list of samples in specific directory
```sh
ls Cell/*.bam | sed -e 's/.bam//g' > cell_sample_list.txt 
```

### Make fq directory 
```sh
mkdir Cell_fq
```

### Convert files to fq
```sh
while read x; do
echo –e “\n… converting ${x} to fastq file …”
bedtools bamtofastq –i Cell/”${x}”.bam –fq Cell_fq/”${x}”.fq
done < cell_sample_list.txt
```

### compress newly made fq files for easier analysis
```sh
while read x; do
bgzip –i Cell_fq/“${x}”.fq && tabix –p fq Cell_fq/“${x}”.fq.gz
done < cell_sample_list.txt
```

### make fastqc directory for samples
```sh
mkdir Cell_fastqc
```

### QC samples with Fastqc
```sh
while read x; do
  echo -e "\n... Processing ${x} - running fastqc ...";
  fastqc --contaminants contaminant_list.txt Cell_fq/"${x}".fq.gz –o Cell_fastqc/;
done <cell_sample_list.txt
```

### contamination
Look at contaminant sequences using NCBI BLAST to determine if actual contamination of just overrepresented sequence 
(if overrepresented add total fasta file of transcript to the contaminants_list.txt and re-run fastqc) these should now 
be labelled with gene Symbol. 

You will notice some reads coming up at ~200bp. These are concatomers created by joining two different transcripts together. 
We will use BBMap to trim the fastq files of these reads for more accurate alignment later on.

```sh
export PATH=$PATH:$HOME/bbmap
while read x; do
echo –e “\n… trimming ${x}.fq.gz of concatomers …”
reformat.sh –Xmx128g in=Cell_fq/”${x}”.fq.gz out=Cell_fq_trimmed/”${x}”_trimmed.fq.gz minlength=50 maxlength=150 overwrite=TRUE
done < cell_sample_list.txt
```

### Using Salmon to quantify transcripts and map them to Ensembl ID’s

#### Build Index 
```sh
/home/miles/software/salmon/Salmon-0.8.2_linux_x86_64/bin/salmon index -t Homo_sapiens.GRCh38.cdna.all.fa.gz -i --type quasi
```

#### salmon alignment: loop through all samples
```sh
while read x; do 
echo -e "\n... Quantifying ${x} with salmon ..."; 
/home/miles/software/salmon/Salmon-0.8.2_linux_x86_64/bin/salmon quant -i hg19_refseq_transcripts/rna_index/ -l A -r Ampliseq/"${x}".fq.gz -p 12 --output Ampliseq_quants/"${x}"_quant;
done <sample_list.txt
```
