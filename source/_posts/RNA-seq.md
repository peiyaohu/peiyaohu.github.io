---
title: RNA-seq
date: 2023-05-29 10:16:18

tags: NGS; RNA-seq; pipeline
---

## 1. Prepare

### a. Create dir and download rawseq && public source 

```sh
#!/bin/bash
mcd() { mkdir "$@" 2> >(sed s/mkdir/mcd/ 1>&2) && cd "$_"; }
### Create working dir:
work_dir=/public/home/pyhu/project/xjw/immunity_mut_RNA_seq
cd $work_dir

# create a dir to save processing files
mcd pyhu_process

# make subdir to save results/scripts/public data source
mkdir 0_Results 2_script 3_source

### PE data are saved at:
seq_dir=/public/home/pyhu/project/xjw/immunity_mut_RNA_seq/rawSeq
```



### b. Download public source

```sh
# create a sub_dir to save public data source and extract downloads (some extracting-scripts are neglected).
cd 3_source; mcd OsativaKitaake
# Downloads info: Transcriptome and hard-masked genome of Kitaake_499 from JGI.
curl --cookie jgi_session=/api/sessions/d3c709055e283ed319a42e6fc3c5fedb --output download.20230529.093434.zip -d "{\"ids\":{\"Phytozome-499\":[\"5a3972e57ded5e35e94f8a26\",\"5a3972e87ded5e35e94f8a30\",\"5a3972e97ded5e35e94f8a32\"]}}" -H "Content-Type: application/json" https://files.jgi.doe.gov/filedownload/

# Transcriptome and hard-masked genoem of Nip(release 56) are also downloaded and saved in the sub_dir (OsNip_release56)
cd ..; mcd OsNip_release56
# transcriptome/annotation/hard-masked genome
wget https://ftp.ensemblgenomes.ebi.ac.uk/pub/plants/release-56/fasta/oryza_sativa/cdna/Oryza_sativa.IRGSP-1.0.cdna.all.fa.gz
wget https://ftp.ensemblgenomes.ebi.ac.uk/pub/plants/release-56/gtf/oryza_sativa/Oryza_sativa.IRGSP-1.0.56.gtf.gz
wget https://ftp.ensemblgenomes.ebi.ac.uk/pub/plants/release-56/fasta/oryza_sativa/dna/Oryza_sativa.IRGSP-1.0.dna_rm.toplevel.fa.gz
```



## 2. RNA-seq quantification using Salmon (v.1.4.0)

### a. Create working dir 

```sh
source_dir=/public/home/pyhu/project/xjw/immunity_mut_RNA_seq/pyhu_process/3_source/OsativaKitaake
cdna_ref=$source_dir/annotation/OsativaKitaake_499_v3.1.transcript.fa.gz
gdna_ref=$source_dir/assembly/OsativaKitaake_499_v3.0.hardmasked.fa.gz
module load salmon/1.4.0
out_dir=/public/home/pyhu/project/xjw/immunity_mut_RNA_seq/pyhu_process/0_Results/4_salmon_kitaake
```



### b. [Salmon-1] Prepare transcriptome indices 

```sh
echo "[1/2] Prepare transcriptome indices (mapping-based mode)" 
# prepare decoy.txt
echo "prepare decoy.txt"
echo "gunzip"

gunzip -c $gdna_ref|grep "^>" | cut -d " " -f 1 > $out_dir/decoys.txt
echo "sed"
bsub -K -J pre-2 -n 2 -R span[hosts=1] -o %J.out -e %J.err -q q2680v2 "sed -i.bak -e 's/>//g' $out_dir/decoys.txt"

# prepare gentrome.fa.gz: the genome targets (decoys) should come after the transcriptome targets in the reference
echo "prepare gentrome.fa.gz"
echo "cat"
bsub -K -J pre-2 -n 2 -R span[hosts=1] -o %J.out -e %J.err -q q2680v2 "cat $cdna_ref $gdna_ref > $out_dir/gentrome.fa.gz"

# Index
echo "Index"
bsub -K -J index -n 2 -R span[hosts=1] -o log/indexkit.out -e log/indexkit.err -q q2680v2 "salmon index -t $out_dir/gentrome.fa.gz \
  -d $out_dir/decoys.txt -p 12 \
  -i $out_dir/salmon_index \
  --gencode"
```



### c. [Salmon-2] Quantifying in mapping-based mode

```sh
echo "[2/2] Quantifying in mapping-based mode"
seq_dir=/public/home/pyhu/project/xjw/immunity_mut_RNA_seq/rawSeq

for i in `ls $seq_dir/*1.fq.gz`;do
gz_index=$(basename $i| sed 's/_1.fq.gz//')

echo "Quantify sample: $gz_index"
bsub -K -J quant_$gz_index -n 2 -R span[hosts=1] -o log/${gz_index}.out -e log/${gz_index}.err -q q2680v2 "salmon quant -i $out_dir/salmon_index \
  -l A \
  -1 $seq_dir/${gz_index}_1.fq.gz \
  -2 $seq_dir/${gz_index}_2.fq.gz \
  --validateMappings \
  -p 6\
  --seqBias \
  -o $out_dir/transcripts_quant"
done
```