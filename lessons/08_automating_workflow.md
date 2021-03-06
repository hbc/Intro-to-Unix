---
title: "Automating an RNA-Seq workflow"
author: "Bob Freeman, Meeta Mistry, Radhika Khetani"
date: "Thursday, May 5, 2016"
---

## Learning Objectives:

* Automate a workflow by grouping a series of sequential commands into a script
* Modify and submit the workflow script to the cluster


### From Sequence reads to Count matrix

That's a lot of work, and you still have five more FASTQ files to go...

- You could try running this workflow on each FASTQ file and then try and combine it all in the end. What would you have to do change in order to get this workflow to work?
- Remembering what commands *and* what parameters to type can be pretty daunting. What can
you do to help yourself out in this regard?
- How do you make sure that you are running every single file with the exact same parametrs, software versions?
- How do you keep track of all the versions and methods? (lab notebook)
- If you were to automate this process, what additional bits of information might you need?


#### Automating this Workflow with a Bash Script

The easiest way for you to be able to repeat this process (from running STAR through to getting counts) is to capture the steps that
you've performed for `Mov10_oe_1` in a bash script. And you've already learned how to do this in previous
lessons. So here's a short exercise...


Using the text editor on your laptop, start creating a script with the commands you used to do the alignment and counting for `Mov10_oe_1`. 

> Note: You can use your command history to retrieve the commands for each step. Don't forget the "shebang line". Make sure you include the `mkdir` commands you used to create new directories.

***
- If this was a script on Orchestra, what would you have to do to run it? 
- In order to run the workflow in this script on another fastq file, you'll need to make changes. _What would have to modify to get this workflow to work with a different file?_

***

#### Granting our Workflow More Flexibility

A couple of changes need to be made to make this script more friendly to both changes in files and changes in the workflow. Let's continue to work on this file on our laptops for now.

##### More variables

**The first major change is allowing flexibility in the input fastq file.** Thus at the start of 
the script let's capture an input parameter that must be supplied on the command line, when running the script.
This input parameter will be the name of the file we want to work on:

    fq=$1

"*The command-line arguments $1, $2, $3,...$9 are positional parameters, with $0 pointing to the actual command, program or shell script, and $1, $2, $3, ...$9 as the arguments to the command.*." 

This basically means:

$0 = name of command/script

$1 = First argument of the command/script

$2 = Second argument of the command/script

$3 = Third argument of the command/script

$* = List of all the positional parameters used

$# = Number of positional parameters

----
For the following script what are $0, $1 and $2?

```bash
sh example_script.sh input1 input2
```

----

> [This is an example](http://steve-parker.org/sh/eg/var3.sh.txt) of a simple script that used the concept of positional parameters and the associated variables. Try writing this script and running it to get a better idea of how these variables are assigned and to bettter understand positional parameters.

Next, we'll initialize variables that contain the paths to where the common files are stored and then use the variable names (with a `$`) in the actual commands later in the script. This is a shortcut for when you want to use this script for a dataset that used a different genome, e.g. mouse; you'll just have to change the contents of these variable at the beginning of the script.

Let's add 2 variables named "genome" and "gtf", these will contain the locations of the genome indices and the annotation file respectively:

    # location of genome reference FASTA and index files + the gene annotation file
    genome=/groups/hbctraining/unix_workshop_other/reference_STAR/
    gtf=~/unix_workshop/rnaseq_project/data/reference_data/chr1-hg19_genes.gtf

Next, make sure you load all the modules for the script to run. This is important so your script can run independent of any "prep" steps that need to be run beforehand:
    
    # set up our software environment...
    module load seq/samtools
    module load seq/htseq

We'll keep the output directory creation, however, we will add the `-p` option this will make sure that `mkdir` will create the directory only if it does not exist, and it won't throw an error if it does exist.

    # make all of our output directories
    # The -p option means mkdir will create the whole path if it 
    # does not exist and refrain from complaining if it does exist
    mkdir -p ~/unix_workshop/rnaseq_project/results/STAR
    mkdir -p ~/unix_workshop/rnaseq_project/results/counts


In the script, it is a good idea to use echo for debugging/reporting to the screen (you can also use `set -x`):

    echo "Processing file $fq ..."

> `set -x` debugging tool will display the command being executed, before the results of the command. In case of an issue with the commands in the shell script, this type of debugging lets you quickly pinpoint the step that is throwing an error. Often, tools will display the error that caused the program to stop running, so keep this in mind for times when you are running into issues where this is not availble.
> The command to turn it off is `set +x`

We also need to extract the "base name" of the file.
```
# grab base of filename for future naming
base=`basename $fq .qualtrim25.minlen35.fq`
echo "basename is $base"
```
> #### Remember `basename`?
> The `basename` command: this command takes a path or a name and trims away all the information before the last `\` and if you specify the string to clear away at the end, it will do that as well. 

Since we've already created our output directories, we can now specify all of our
output files in their proper locations. We will assign various file names to
 variables both for convenience but also to make it easier to see what 
is going on in the command below.
```
# set up output filenames and locations
align_out=~/unix_workshop/rnaseq_project/results/STAR/${base}_
counts_input_bam=~/unix_workshop/rnaseq_project/results/STAR/${base}_Aligned.sortedByCoord.out.bam
counts=~/unix_workshop/rnaseq_project/results/counts/${base}.counts
```
Our variables are now staged. We now need to modify the series of commands starting with STAR through to counts (htseq-count)
to use these variables so that it will run the steps of the analytical workflow with more flexibility:

    # Run STAR
    STAR --runThreadN 6 --genomeDir $genome --readFilesIn $fq --outFileNamePrefix $align_out --outFilterMultimapNmax 10 --outSAMstrandField intronMotif --outReadsUnmapped Fastx --outSAMtype BAM SortedByCoordinate --outSAMunmapped Within --outSAMattributes NH HI NM MD AS

    # Create BAM index
    samtools index $counts_input_bam

    # Count mapped reads
    htseq-count --stranded reverse --format bam $counts_input_bam $gtf  >  $counts

It is always nice to have comments at the top of a more complex script to make sure that when your future self, or a co-worker, uses it they know exactly how to run it and what the script will do. So for our script, we can have the following lines of comments right at the top after `#!/bin/bash/`:

```
# This script takes a trimmed fastq file of RNA-Seq data and outputs a counts file for it.
# USAGE: sh rnaseq_analysis_on_allfiles.sh <name of fastq file>
```

Use `pwd` to check what your current directory is, and make sure that you are in the `~/unix_workshop/rnaseq_project/` directory. Once you save this script (`rnaseq_analysis_on_input_file.sh`) in the `~/unix_workshop/rnaseq_project/` directory, and it is ready for running. 
> **To transfer the saved file to Orchestra, you can either copy and paste the script as a new `nano` file, or use Filezilla.**
> 
> `$ pwd`
>
> `$ nano rnaseq_analysis_on_input_file.sh`

Once the script has been saved, make it executable before running it. This is good to do even if your script runs fine without it; it will help avoid any future problems, and will enable your future self to know that it's an executable shell script.

```
$ chmod u+rwx rnaseq_analysis_on_input_file.sh 

$ sh rnaseq_analysis_on_input_file.sh <name of fastq>
```

#### Running our script iteratively as a job submission to the LSF scheduler

**The above script will run in an interactive session one file at a time. If we wanted to run this script as a job submission to LSF, and with only one command have LSF run through the analysis for all your input fastq files?**

To run the above script iteratively for all of the files on a worker node via the job scheduler, we need to create a **new submission script** that will need 2 important components:

1. our **LSF directives** at the **beginning** of the script. This is so that the scheduler knows what resources we need in order to run our job on the compute node(s).
2. a for loop that iterates through and runs the above script for all the fastq files.

Let's create a new file with nano and call it `rnaseq_analysis_on_allfiles.lsf`:

	$ nano rnaseq_analysis_on_allfiles.lsf

> Please note that the extension on this script is `lsf`, but it can be anything you want. The reason it's a good idea to have submission scripts on orchestra have this extension is, once more, for your future self to know right away what is possible in this script, i.e. a set of commands preceded by LSF directives.

The top of the file should look like with the LSF directives:

    #!/bin/bash

    #BSUB -q priority		# Partition to submit to (comma separated)
    #BSUB -n 6                  # Number of cores, since we are running the STAR command with 6 threads
    #BSUB -W 1:30               # Runtime in D-HH:MM (or use minutes)
    #BSUB -R "rusage[mem=4000]"    # Memory in MB
    #BSUB -J rnaseq_mov10         # Job name
    #BSUB -o %J.out       # File to which standard out will be written
    #BSUB -e %J.err       # File to which standard err will be written

	# this `for` loop, will take our trimmed fastq files as input and run the script for all of them one after the other. 

    for fq in ~/unix_workshop/rnaseq_project/data/trimmed_fastq/*.fq
    do
      sh rnaseq_analysis_on_input_file.sh $fq
    done

Before you run this script, let's add a few more commands after the `for` loop for creating a count matrix. These commands will be executed after all the files have been processed *serially* through the `rnaseq_analysis_on_input_file.sh` script:

    # define a variable that has the name of the file and path to the final count matrix
    countmatrix=results/counts/Mov10_rnaseq_counts.txt

    # Concatenate all count files into a single count matrix using paste, awk and grep along with the awesomeness of pipes!
    paste results/counts/Mov10_oe_1.counts results/counts/Mov10_oe_2.counts results/counts/Mov10_oe_3.counts results/counts/Irrel_kd_1.counts results/counts/Irrel_kd_2.counts results/counts/Irrel_kd_3.counts | awk '{print$1"\t"$2"\t"$4"\t"$6"\t"$8"\t"$10"\t"$12}' | grep -v "^__" > $countmatrix


Our submission script is now complete; save and exit out of nano and submit away (note the `<` between the command and the script name):
```
$ bsub < rnaseq_analysis_on_allfiles.lsf
```

> ##### Processing the count matrix for downstream applications
>
> Now we have a count matrix for our dataset, the only thing we are missing is a header to indicate which columns correspond to which sample. We can add it, "ID OE.1 OE.2 OE.3 IR.1 IR.2 IR.3", using `nano` or another text editor. 

#### Parallelizing workflow for efficiency

**The above script will run through the analysis for all your input fastq files, but it will do so in serial. We can set it up so that the pipeline is working on all the trimmed data in parallel (at the same time). This will save us a lot of time when we have realistic datasets.**

Let's make a modified version of the above script to parallelize our analysis. To do this need to modify one major aspect which will enable us to work with some of the contrainsts that this scheduler (LSF) has. We will be using a for loop for submission and putting the directives for each submission in the bsub command.

Let's make a new file called `rnaseq_analysis_on_allfiles-for_lsf.sh`. Note this is a normal shell script.


	$ nano rnaseq_analysis_on_allfiles_for-lsf.sh


This file will loop through the same files as in the previous script, but the command it submits will be the actual bsub command:

```
#! /bin/bash
for fq in ~/unix_workshop/rnaseq_project/data/trimmed_fastq/*.fq
do
bsub -q priority -n 6 -W 1:30 -R "rusage[mem=4000]" -J rnaseq_mov10 -o %J.out -e %J.err sh rnaseq_analysis_on_input_file.sh $fq
sleep 1
done
```

	sh rnaseq_analysis_on_allfiles_for-lsf.sh

**In the context of this script, you will have to run the command (`paste ...`) to put the count matrix with all the counts files together after all the jobs finished running.**

> NOTE: All job schedulers are similar, but not the same. Once you understand how one works, you can transition to another one without too much trouble. They all have their pros and cons that the system administrators for your setup have taken into consideration and picked one that fits the needs of the users best. 

What you should see on the output of your screen would be the jobIDs that are returned
from the scheduler for each of the jobs that your script submitted.

You can see their progress by using the `bjobs` command (though there is a lag of
about 60 seconds between what is happening and what is reported).

Don't forget about the `bkill` command, should something go wrong and you need to
cancel your jobs.

---
*To share or reuse these materials, please find the attribution and license details at [license.md](https://github.com/hbc/Intro-to-Unix/blob/master/license.md).*
