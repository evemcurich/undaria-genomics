# undaria-genomics
scripts and things to do with the bioinformatics side of processing different samples of undaria pinnatifida genomes 
all code was run as batch scripts within the HPC at Newcastle University, "Rocket".

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

















