#!/bin/bash

#BSUB -q priority   # queue name
#BSUB -W 2:00       # hours:minutes runlimit after which job will be killed.
#BSUB -n 6      # number of cores requested
#BSUB -J rnaseq_mov10_qc         # Job name
#BSUB -o %J.out       # File to which standard out will be written
#BSUB -e %J.err       # File to which standard err will be written

# Change directories into the folder with the untrimmed fastq files
cd ~/unix_workshop/rnaseq_project/data/untrimmed_fastq

# Loading modules for tools
module load seq/Trimmomatic/0.33
module load seq/fastqc/0.11.3

# Run Trimmomatic
echo "Running Trimmomatic..."
for infile in *.fq
do

  # Create names for the output trimmed files
  base=`basename $infile .subset.fq`
  outfile=$base.qualtrim25.minlen35.fq

  # Run Trimmomatic command
  java -jar /opt/Trimmomatic-0.33/trimmomatic-0.33.jar SE \
  -threads 4 \
  -phred33 \
  $infile \
  ../trimmed_fastq/$outfile \
  ILLUMINACLIP:/opt/Trimmomatic-0.33/adapters/TruSeq3-SE.fa:2:30:10 \
  TRAILING:25 \
  MINLEN:35

done

# Run FastQC on all trimmed files
echo "Running FastQC..."
fastqc -t 6 ../trimmed_fastq/*.fq
