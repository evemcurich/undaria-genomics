# undaria-genomics
scripts and things to do with the bioinformatics side of processing different samples of undaria pinnatifida genomes 
all code was run as batch scripts within the HPC at Newcastle University, "Rocket".

It's important throughout this to check the directory you're working in, as well as making sure when you run commands that you specify to the script where the file you're aiming at is located, to avoid it failing

## 1.0 Using Conda
### 1.1 Creating a conda environment
  It's useful to have separate conda environments for different stages of genome processing. Sometimes, when you install many packages into one environment, they can break and not run as expected.
```
conda create --name <env-name>
```
### 1.2 Activating a conda environment
  You will need to repeat this step every time you open a new terminal
```
conda activate <env-name>
```
  To know if you successfully activated the environment, the (base) which usually proceeds your working directory should be changed to the name of your activated environment

#### 1.2.1 Deactivating a conda environment
If you need to switch to a different environment, you can first deactivate the one you are in, then repeat 1.3 to activate your new environment
```
conda deactivate
```
This command does not require you to name the environment you are currently in.

#### 1.2.2 Listing your conda environments
This allows you to chek the names of your created environments, if you forget their names
```
conda env list
```
#### 1.2.3 Removing a conda environment
If you create a conda environment you no longer need, you can remove it
```
conda env remove --name <environment-name>
```

### 1.3 Installing conda packages within an environment
  Locate the line of code to install your desired by packaging by googling "<package> for anaconda".
  For example, one of the packages you will need throughout this process is SRA-Tools
  <img width="1045" alt="Screenshot 2024-07-23 at 10 35 16" src="https://github.com/user-attachments/assets/09b188e3-377f-4b03-a75a-7e5a2372b47f">
  From this page on the Anaconda website, all we need to do is pull one of the lines of 'conda install' code at the bottom of the page
```
conda install bioconda::sra-tools
```
  As this code runs, select 'y' or 'yes' to any file changes, and your package will successfully install.

## 2.0 Processing Sequence Reads
### 2.1 Downloading the sequence reads
Sequence Read Archive (SRA) files for the Undaria Pinnatifida genome can be downloaded from this location:
https://www.ncbi.nlm.nih.gov/bioproject/?term=PRJNA646283
### 2.2 Converting SRA > fastq
SRA files contain the data we need, but not a format which we can work on until we 'dump' it - so we need to convert it to fastq.
To do this, first download the package 'SRA-Tools' from https://anaconda.org/bioconda/sra-tools into your *activated* environment
### 2.2.1 Downloading SRA-Tools
```
conda install bioconda::sra-tools
```
### 2.2.2 fastq-dump
```
fastq-dump --split-3 <ID>.sra
```
The '--split-3' option will produce 2 files for each .SRA file, the forward and reverse sequence for each genome
### 2.2.3 Setting up an array job
To make life easier, rather than submitting a separate job script for every single SRA file, you can write one script which will make many smaller jobs for you.
This makes use of the Python 'dead simple queue' for SLURM on HPC systems.
dSQ: https://github.com/ycrc/dSQ
1. From this repository, download the 'dSQ.py' and 'dSQBatch.py' files, and place them into the same directory.
2. Create a script file (.sh) which will use dSQ to generate a list of jobs to carry out as an array. I'd recommend naming the job and output of this one something to indicate that this is the script which will make another text file, not actually send your jobs off.
```
nano <NAME>txtscript.sh
```
the command 'nano' just opens a text editor which will allow you to type up/paste in your batch scripts. Paste the below into your nano file, then pressed ctrl+x to close and y to save.
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=<NAME>-txtscript 
#SBATCH --output=<NAME>_%A_%a.out
#SBATCH --error=<NAME>_%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

for x in *.SRA; do echo "fastq-dump --split-3 $x" >> dump.txt; done
```
this code basically tells our loop to repeat the command for each job, each time replacing the x for any file ending in .SRA
the * incidates anything ending in .SRA
  
3. Next, create a dSQ script which will put these separate commands into one job script together. I'd also recommend naming this one something to indicate its your actual queue script that will send off your jobs. I usually do this by adding dsq- at the start.
```
nano dsq-dump.sh
```
Now paste this into your nano window!
however, there are a few things you can change:
-p to change the system of cores(?) you're using
--array can be changed in accordance to the number of files you are working on. Since mine will generate 36 different jobs for 36 different SRA files, I set my array as 0-35
  -> (0 counts as the first one)
the "60g" can also be changed to however much memory you want to use, but most systems will have a set min/max
-t can be changed if you'd like your jobs to run for longer. "1-00:00:00" is a day, but you can change this
--mail-type "ALL" will automatically email you when your jobs have begun, when they finish, or if they fail
```
#!/bin/bash
#SBATCH --output dsq-<NAME>_%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name <NAME>
#SBATCH -p defq
#SBATCH --mem-per-cpu "60g" -t "1-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /path/to/directory/dSQBatch.py --job-file /path/to/directory/dump.txt --status-dir /path/to/directory
```
4. Run the dsQ script!
```
sbatch dsq-<NAME>.sh
```
5. If you get a failed message from SLURM, you can check the error and output files to see what the problem might be.
```
nano *.err
nano *.out
```
To make things easier, you can move all of your new .fastq files to a new folder to work on them there, to make sure it's not just one directory getting too full
```
mv *.fastq fastq
```
This line of code would be moving any files ending in .fastq to the new fastq directory.
### 2.3 Checking read quality
Observing the quality of the raw reads you have is an important step before you move on to trimming for the pieces of DNA which are useful. Doing this step will allow you to edit the parameters of the trimming stage, if the strictness of what to cut needs to be changed.
This stage uses the packages 'fastQC' - https://anaconda.org/bioconda/fastqc and 'multiqc' - https://anaconda.org/bioconda/multiqc

1. Install fastqc
```
conda install bioconda::fastqc
conda install conda install bioconda::multiqc
```
2. Adapt the previous initial dSQ command to carry out fastqc instead
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=<NAME>
#SBATCH --output=<NAME>_%A_%a.out
#SBATCH --error=<NAME>_%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

for x in *.fastq.gz; do echo "fastqc $x" >> fastqc.txt; done
```
3. Adapt your actual dSQ send-off command to the new txt file, by changing the name of the text file at the end of the --job-file command
```
#!/bin/bash
#SBATCH --output dsq-<NAME>_%A_%2a-%N.out
#SBATCH --array 0-70
#SBATCH --job-name <NAME>
#SBATCH -p defq
#SBATCH --mem-per-cpu "20g" -t "1-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /path/to/directory/dSQBatch.py --job-file /path/to/directory/fastqc.txt --status-dir /path/to/directory
```
The fastqc command will generate many .html reports. When you are working with lots of these, it is easier to use multiqc to view them all simulataneously for comparsion.
5.  multiqc to visualise all of the results
it's recommended to make a new conda environment just for multiqc, as it can be a bit tempermental with other packages trying to overwrite each other
```
multiqc /path/to/directory
```
This command will create one html file which you can analyse for read quality.

## 2.4 Trimming your reads
This section will use 'Trim_Galore'. Either switch back to your first environment, or make a new one for trimming.
https://anaconda.org/bioconda/trim-galore
### 2.4.1 Using dSQ
For this step, it's again easier to create a dSQ which will perform the same command on all of the different reads simultaneously to save time. However, this time we'll change the original script slightly to 'pull' the SRA IDs from a list to make the pairing situation easier.
1. Create a "samples.txt" file, containing all of your SRA IDs in order
```
nano "samples.txt"
```
You would paste a list of all of your SRA file names into here, but without the .sra extension on the end -- just the SRA followed by all the numbers!

2. Adapt your initial dsq script file, with a few new additions
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=<NAME>
#SBATCH --output=<NAME>_%A_%a.out
#SBATCH --error=<NAME>_%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

filename="samples.txt"
for x in $(cat "$filename"); do echo "trim_galore --paired "$x"_1.fastq.gz "$x"_2.fastq.gz" >> trim_all.txt; done
```
"filename =" establishes where your list of IDs is
the $(cat "$filename") will tell the script to read each line of the txt file and subtitute the $x in the actual command for the SRA ID, followed by the correct file name

3. Again, send off your new dSQ as an actual array job
```
#!/bin/bash
#SBATCH --output dsq-<NAME>_%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name <NAME>
#SBATCH -p defq
#SBATCH --mem-per-cpu "20g" -t "1-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /path/to/directory/dSQBatch.py --job-file /path/to/directory/trim_all.txt --status-dir /path/to/directory
```
This will give you your new trimmed files!
## 3.0 Read Alignment
This stage will guide you through aligning your newly trimmed reads to a reference genome. The packages used will be bowtie2, Sam-Tools, Bam-Tools and Qualimap
https://anaconda.org/bioconda/bowtie2
https://anaconda.org/bioconda/samtools
https://anaconda.org/bioconda/bamtools
https://anaconda.org/bioconda/qualimap

It'll also require the use of a reference genome, obtained from:
https://www.ncbi.nlm.nih.gov/datasets/genome/GCA_012845835.1/

## 3.1 Indexing the reference genome
bowtie2 is our first package to be used to index the reference genome
We won't need to use dSQ for this stage, since there's only one file we're working on.
So, you can just put this straight into a script file using nano and send it off!
```
nano genomeindex.sh
```
Then paste the following...
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=genomeindex
#SBATCH --output=genomeindex_out_%A_%a.out
#SBATCH --error=genomeindex_err_%A_%a.err
#SBATCH -p interactive
#SBATCH --time=1-00:00:00
#SBATCH --mem=40G
#SBATCH --mail-type=ALL

bowtie2-build GCA_012845835.1_ASM1284583v1_genomic.fna undaria_index
```
Here I named it undaria_index for ease, and it is referred to that throughout the rest of the code.
This will create 6 files with the extension .bt2

## 3.2 Aligning the reads
1. First of all, to put our files somewhere easily accessible and away from everything we've previously been working on, I would recommend making a new directory (or two) inside the one you've already been working in. Here I will create a new directory called bowtie, with two other files inside called sam and bam, as those are the names of the packages being used.
```
mkdir bowtie
mkdir bowtie/sam
mkdir bowtie/bam
```
2. We'll repeat again the creating of the dsq file, using our same samples text file from before to list all of our different SRA IDs. Not much changes apart from the bowtie2 command itself! I've named this job file bowtie2 since that is the command we're running, but you can change it to anything to make it easily identifiable
The only thing to double check is that you are still working from the directory where your samples.txt file is, otherwise you may need to move it
```

#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=b
#SBATCH --output=bowtie2_%A_%a.out
#SBATCH --error=bowtie2_%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

filename="samples.txt"
for x in $(cat "$filename"); do echo "bowtie2 -x /path/to/directory/undaria_index -1 "$x"_1_val_1.fq.gz -2 "$x"_2_val_2.fq.gz -S /path/to/directory/fastq/bwa/sam/"$x".sam" >> align_all.txt; done

```
2. Again, run the second dsq file with the new align_all.txt file
-> Change the array number if you need to, or the time allowed if yours will take longer/shorter
```
#!/bin/bash
#SBATCH --output dsq-bwamemALL-%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name bowtie
#SBATCH -p long
#SBATCH --mem-per-cpu "20g" -t "3-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /path/to/directory/dSQBatch.py --job-file path/to/directory/align_all.txt --status-dir /path/to/directory

```
This will result in us having one .sam file for each SRA ID now in your SAM directory.
Now we can convert them to BAM!

## Converting SAM to BAM
This section will only require SamTools
https://anaconda.org/bioconda/samtools
Or just enter this code into your activated environment:
```
conda install bioconda::samtools
```














