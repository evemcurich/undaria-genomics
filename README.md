# undaria-genomics
scripts and things to do with the bioinformatics side of processing different samples of undaria pinnatifida genomes 
all code was run as batch scripts within the HPC at Newcastle University, "Rocket".

## 1.1 Using Conda
### 1.2 Creating a conda environment
  It's useful to have separate conda environments for different stages of genome processing. Sometimes, when you install many packages into one environment, they can break and not run as expected.
```
conda create --name "env-name"
```
### 1.3 Activating a conda environment
  You will need to repeat this step every time you open a new terminal
```
conda activate <env-name>
```
  To know if you successfully activated the environment, the (base) which usually proceeds your working directory should be changed to the name of your activated environment

### 1.4 Deactivating a conda environment
```
conda deactivate
```
This command does not require you to name the environment you are currently in.

### 1.5 Installing conda packages within an environment
  Locate the line of code to install your desired by packaging by googling "<package> for anaconda".
  For example, one of the packages you will need throughout this process is SRA-Tools
  <img width="1045" alt="Screenshot 2024-07-23 at 10 35 16" src="https://github.com/user-attachments/assets/09b188e3-377f-4b03-a75a-7e5a2372b47f">
  From this page on the Anaconda website, all we need to do is pull one of the lines of 'conda install' code at the bottom of the page
```
conda install bioconda::sra-tools
```
  As this code runs, select 'y' or 'yes' to any file changes, and your package will successfully install.

