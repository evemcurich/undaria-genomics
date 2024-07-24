# undaria-genomics
scripts and things to do with the bioinformatics side of processing different samples of undaria pinnatifida genomes 
all code was run as batch scripts within the HPC at Newcastle University, "Rocket".

It's important throughout this to check the directory you're working in, as well as making sure when you run commands that you specify to the script where the file you're aiming at is located, to avoid it failing

# 1.0 Using Conda
#### 1.1 Creating a conda environment
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

### 1.2.1 Deactivating a conda environment
If you need to switch to a different environment, you can first deactivate the one you are in, then repeat 1.3 to activate your new environment
```
conda deactivate
```
This command does not require you to name the environment you are currently in.

### 1.2.2 Listing your conda environments
This allows you to chek the names of your created environments, if you forget their names
```
conda env list
```
### 1.2.3 Removing a conda environment
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
2. Create a script file (.sh) which will use dSQ to generate a list of jobs to carry out as an array
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

for x in *.SRA; do echo "fastq-dump --split-3 $x" >> dump.txt; done
```
3. Next, create a dSQ script which will put these separate commands into one job script together
```
nano dsq-dump.sh
```
```
#!/bin/bash
#SBATCH --output dsq-<NAME>_%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name <NAME>
#SBATCH -p long
#SBATCH --mem-per-cpu "20g" -t "3-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /path/to/directory/dSQBatch.py --job-file /path/to/directory/dump.txt --status-dir /path/to/directory
```
4. Run the dsQ script!
```
sbatch <NAME>.sh
```
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
3. Adapt your actual dSQ send-off command to the new txt file
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
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=indexingBT
#SBATCH --output=indexBT_out_%A_%a.out
#SBATCH --error=indexBT_err_%A_%a.err
#SBATCH -p interactive
#SBATCH --time=1-00:00:00
#SBATCH --mem=40G
#SBATCH --mail-type=ALL

bowtie2-build GCA_012845835.1_ASM1284583v1_genomic.fna undaria_index
```
Here I named it undaria_index for ease, and it is referred to that throughout the rest of the code.
This will create 6 files with the extension .bt2

## 3.2 Aligning the reads
1. We'll repeat again the creating of the dsq file, using our same samples text file from before to list all of our different SRA IDs. Not much changes apart from the bowtie2 command itself!
```

#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=bowtie2
#SBATCH --output=bowtie2_%A_%a.out
#SBATCH --error=bowtie2_%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

filename="samples.txt"
for x in $(cat "$filename"); do echo "bowtie2 -x /nobackup/proj/ejwg/Eve_M_Proj/SRA_download/undaria_index -1 "$x"_1_val_1.fq.gz -2 "$x"_2_val_2.fq.gz -S /nobackup/proj/ejwg/Eve_M_Proj/SRA_download/fastq/bwa/sam/"$x".sam" >> align_all.txt; done

```
2. Again, run the second dsq file with the new align_all.txt file
```
#!/bin/bash
#SBATCH --output dsq-bwamemALL-%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name bowtie
#SBATCH -p long
#SBATCH --mem-per-cpu "20g" -t "3-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /nobackup/proj/ejwg/Eve_M_Proj/dsq/dSQBatch.py --job-file /mnt/storage/nobackup/proj/ejwg/Eve_M_Proj/SRA_download/align_all.txt --status-dir /mnt/storage/nobackup/proj/ejwg/Eve_M_Proj/SRA_download

```
















