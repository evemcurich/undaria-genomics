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
2. Create a samples.txt file which just contains all of your sample IDs without any file extensions, i.e. without the .fq
```
nano samples.txt
```
3. Create a script file (.sh) which will use dSQ to generate a list of jobs to carry out as an array. I'd recommend naming the job and output of this one something to indicate that this is the script which will make another text file, not actually send your jobs off.
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

filename="samples.txt"
for x in $(cat "$filename"); do echo "fastq-dump --split-3 $x" >> dump.txt; done
```
this code basically tells our loop to repeat the command for each job, each time replacing the x for any file ending in .SRA
"samples.txt" contains all of our sample IDs, without any file extensions
  
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

filename="samples.txt"
for x in $(cat "$filename"); do echo "fastqc $x" >> fastqc.txt; done
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
## 3.0 Read Alignment and Filtering
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
3. Again, run the second dsq file with the new align_all.txt file
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

## 3.3 Converting SAM to BAM
This section will only require SamTools
https://anaconda.org/bioconda/samtools
Or just enter this code into your activated environment:
```
conda install bioconda::samtools
```
We can once again set this up as a queue for if you are working with many samples
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=conversion
#SBATCH --output=conversion_%A_%a.out
#SBATCH --error=conversion_%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

filename="samfiles.txt"
for x in $(cat "$filename"); do echo "samtools view -bh ./bwa/sam/"$x".sam > ./bwa/bam/"$x".bam" >> convert.txt; done
```
Followed by the queue executioning script, as usual
Ensure the directory leading to the script text file is correct from where you created it in the first step
```
#!/bin/bash
#SBATCH --output dsq-convert-%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name converting
#SBATCH -p long
#SBATCH --mem-per-cpu "20g" -t "3-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /nobackup/proj/ejwg/Eve_M_Proj/dsq/dSQBatch.py --job-file /path/to/directory/convert.txt --status-dir /path/to/directory
```
## 3.4 Sorting and Indexing the BAM files
Now we've made our BAM files, we need to sort and index them again. You can either run this as a queue, or separately, depending on how many samples you have.
1) As a queue:
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=sortindex
#SBATCH --output=sortindex_%A_%a.out
#SBATCH --error=sortindex_%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

filename="samples.txt"
for x in $(cat "$filename"); do echo "samtools sort -o "$x"_sorted.bam "$x".bam
                          samtools index "$x"_sorted.bam" >> sortindex.txt; done
```
Then run the queue script
```
#!/bin/bash
#SBATCH --output dsq-bamindex-%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name bamindex
#SBATCH -p defq
#SBATCH --mem-per-cpu "60g" -t "2-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /nobackup/proj/ejwg/Eve_M_Proj/dsq/dSQBatch.py --job-file /path/to/directory/sortindex.txt --status-dir /path/to/directory/
```
## 3.5 Filter the alignment
We now filter the BAM file to remove reads which are of a low quality. This means that reads which have a mapping quality below 30, are not non-primary alignments, and are pair-ended but mapped more than 800bp apart, will be removed from the sequence. We then re-index these files again.
1. Make the first dsq file
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=filterbam
#SBATCH --output=filterbam_%A_%a.out
#SBATCH --error=filterbam_%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

filename="samples.txt"
for x in $(cat "$filename"); do echo "bamtools filter -mapQuality '>=30' -isPrimaryAlignment 'true' -insertSize '<=800' -in "$x"_sorted.bam -out "$x"_sorted.mq30.maxinsert.primaryalignment.bam

  samtools index "$x"_sorted.mq30.maxinsert.primaryalignment.bam" >> filterbam.txt; done
```
2. Make the dsq-executable
```
#!/bin/bash
#SBATCH --output dsq-filterbam-%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name filterbam
#SBATCH -p defq
#SBATCH --mem-per-cpu "60g" -t "2-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /nobackup/proj/ejwg/Eve_M_Proj/dsq/dSQBatch.py --job-file /path/to/directory/filterbam.txt --status-dir /path/to/directory/
```
## 3.6 Mark PCR Duplicates
Now we want to remove duplicates of the same sequences to declutter our DNA
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=pcrremoval
#SBATCH --output=pcrremoval%A_%a.out
#SBATCH --error=pcrremoval%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

filename="samples.txt"
for x in $(cat "$filename"); do echo "java -jar ~/Share/picard.jar MarkDuplicates \
      I=SRR518707_sorted.mq30.maxinsert.primaryalignment.bam \
      O=SRR518707_sorted.mq30.maxinsert.primaryalignment_marked_duplicates.bam \
      M=SRR518707_marked_dup_metrics.txt" >> pcrremoval.txt; done
```
Execute!
```
#!/bin/bash
#SBATCH --output dsq-pcrremoval-%A_%2a-%N.out
#SBATCH --array 0-35
#SBATCH --job-name pcrremoval
#SBATCH -p defq
#SBATCH --mem-per-cpu "60g" -t "2-00:00:00" --mail-type "ALL"

# DO NOT EDIT LINE BELOW
python /nobackup/proj/ejwg/Eve_M_Proj/dsq/dSQBatch.py --job-file /path/to/directory/pcrremoval.txt --status-dir /path/to/directory/
```
## 4.0 VCF making
Now we've got our final BAM file, we can convert it into a vcf to filter it down further, then actually use it for some anaylsis
To do this, we'll be using bcftools again, with a couple of important flags. For this command too, you won't need to make a queue, as it will be compiling very BAM file you have into one single vcf file. 
1. Making the VCF
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=vcfmaking
#SBATCH --output=vcf%A_%a.out
#SBATCH --error=vcf%A_%a.err
#SBATCH -p long
#SBATCH --time=5-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

bcftools mpileup -f /path/to/directory/GCA_012845835.1_ASM1284583v1_genomic.fa --annotate AD,DP -b /path/to/directory/all.bamlist | bcftools call -m -v -f GQ --skip-variants indels -Oz -o /path/to/directory/50inds.vcf.gz
```
To break this command down: 
'-f' is a flag for us to insert our reference genome we'll align all of the BAM files to to make one vcf
'--annotate AD,DP' tells the output to include both the allele depth (AD) and total depth (DP)
'-b' tells the command to refer to the bamlist for a list of files to compile into this vcf. 
'-m' tells the command to use the multi-allelic version of the genotype caller
'-v' only prints variant sites, which condenses how many reads we end up with
'-f' is a second call command which allows us to also add GQ to assess genotype quality score
'--skip-variants indels' allows us to remove indels, as in this case we're only interested in SNPS -- (Single nucleotide polymorphisms)
'Oz -o' specifies our output file name, and also that we'd like it zipped using gzip too to further condense file size. You can remove the Oz if you don't mind having a large-ish vcf. 

FOR THE BAMLIST:
The bamlist should be a list of all files, including their full extensions when we their final bam stage, i.e. in this case should all end in '_sorted.mq30.maxinsert.primaryalignment_marked_duplicates.bam'. To make sure the command understands it fully also, I'd also include the directory at the beginning to ensure it isn't looking for them in some weird location. As an example, it should look something like this:
  '/path/to/directory/SRR12223717_sorted.mq30.maxinsert.primaryalignment_marked_duplicates.bam'

For this one, in terms of time, I'd give it a WHILE. This is a huge process, and for 52 individuals took about 3 days for me. 
This command should run to give you a single VCF file. If you perform the 'nano' command on the .err and .out files, they should inform you how many individuals (hopefully them all!) were inserted into the vcf, so you can double check none were missed. 

## 5.0 VCF filtering
Before starting any analysis, it's a good idea to remove any poorly supported data from the vcf. We can do this through SNP filtering, primarily using vcftools
https://anaconda.org/bioconda/vcftools
### 5.1 Depth and Genotype Quality
The first filter to run is for depth and genotype quality. This might need some adjusting depending on the quality of your data, and you can analyse whether or not the filter was approriate by how many sites were left remaining after conducting the filter. You can find this out by looking at the .out and .err files again after running the job. 
Again too, you won't need any queues from now on - all this is one file!
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=DP_GQ
#SBATCH --output=DP_GQ%A_%a.out
#SBATCH --error=DP_GQ%A_%a.err
#SBATCH -p defq
#SBATCH --time=2-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

vcftools --gzvcf 50inds.vcf.gz --minDP 6 --minGQ 18 --recode --out 50inds_minDP6GQ18
```
Then, do nano of the .err and .out to check if the filtering was appropriate! If no bases have been removed at this point - that's fine. If loads have been taken away, you could lower both filters by a few numbers.
### 5.2 Biallelic Loci Retention
This filter will ensure we don't have any loci which only have one allele. We do this by making sure each site has a max of 2 and a min of 2 alleles.
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=biallelic
#SBATCH --output=biallelic%A_%a.out
#SBATCH --error=biallelic%A_%a.err
#SBATCH -p defq
#SBATCH --time=2-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

vcftools --vcf 50inds_minDP6GQ18.recode.vcf --max-alleles 2 --min-alleles 2 --recode --out 50inds_minDP6GQ18_biallelic
```
### 5.3 Missing Data Removal
This filter allows to remove sites which have 'too much' missing data. In this case, we've used 0.75 to represent us allowing to keep 25% of missing data - meaning for 50 individuals, ~12 individuals can be missing a genotype call. 
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=missingdata
#SBATCH --output=missingdata%A_%a.out
#SBATCH --error=missingdata%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

vcftools --vcf 50inds_minDP6GQ18_biallelic.recode.vcf --max-missing 0.75 --recode --out 50inds_minDP6GQ18_biallelic_0.25missing
```
### 5.4 Minor Allele Count/Minor Allele Frequency
This is another common filter used to remove loci where alleles appear very infrequently -- i.e. to take out any data which is extremely rare and would affect the analysis results for no good reason. In this case we use a MAC of 4, meaning minor alleles have to appear at least 4 times in order to be retained.
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=MAC4
#SBATCH --output=MAC4%A_%a.out
#SBATCH --error=MAC4%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

vcftools --vcf 50inds_minDP6GQ18_biallelic_0.25missing.recode.vcf --mac 4 --recode --out 50inds_minDP6GQ18_biallelic_0.25missing_mac4
```
### 5.5 Max mean depth
We've already done minimum depth -- now we need to also make sure no sites go *too* deep also. We can do this first by generating a report which will tell us the mean site depth across the VCF
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=mdreport
#SBATCH --output=mdreport%A_%a.out
#SBATCH --error=mdreport%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL
vcftools --vcf 50inds_minDP6GQ18_biallelic_0.25missing_mac4.recode.vcf --site-mean-depth --out 50inds_minDP6GQ18_biallelic_0.25missing_mac4
```
Note how we don't recode this one as we're not changing anything yet - we're just generating a report!
The report should be a .ldepth.mean file -- you can open it using a text editor to read it.
The best way to figure out which value you should actually use as your maximum depth is by inputting the dataset into R, and figuring out mean and SD
```
R
ldepth<-read.table("50inds_minDP6GQ18_biallelic_0.25missing_mac4.ldepth.mean", sep="\t", header = TRUE)
mean(ldepth$MEAN_DEPTH)
sd(ldepth$MEAN_DEPTH)
q()
```
My results from this gave me a max mean depth to use of 20, but you can edit it to whatever R gives you

Now, we just need to run the filter
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=max20
#SBATCH --output=max20%A_%a.out
#SBATCH --error=max20%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

vcftools --vcf 50inds_minDP6GQ18_biallelic_0.25missing_mac4.recode.vcf --max-meanDP 20 --recode --out 50inds_minDP6GQ18_biallelic_0.25missing_mac4_maxDP
```
## 5.6 Linkage Disequilibrium
Here, we want to 'thin' our loci to only maintain sites which are a specific distance apart from each other with the reference genome contigs. The simplest method of doing this is to just insert a typical distance -- in this case 1 locus per 10kb. 
You can look into LD and figure out if another value is more appropriate
```
#!/bin/bash
#SBATCH -c 8
#SBATCH -t 0-10
#SBATCH --job-name=ld
#SBATCH --output=ld%A_%a.out
#SBATCH --error=ld%A_%a.err
#SBATCH -p defq
#SBATCH --time=1-00:00:00
#SBATCH --mem=60G
#SBATCH --mail-type=ALL

vcftools --vcf 50inds_minDP6GQ18_biallelic_0.25missing_mac4.recode.vcf --thin 10000 --recode --out 50inds_minDP6GQ18_biallelic_0.25missing_mac4_thinned10kb
```
This was our final filter! If we're happy with the amount of sites remaining, we can then move on to analysis - but this is it for the bioinformatics portion!






