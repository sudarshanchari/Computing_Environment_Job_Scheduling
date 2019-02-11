# Computing Environment and Job Scheduling
My general strategy for using virtual environments and submitting jobs to a queue via schedulers

## Environment
- I like doing much of the sequencing/ scripting related projects in a conda environment (https://conda.io/projects/conda/en/latest/user-guide/getting-started.html). (for a few reasons why, see https://medium.freecodecamp.org/why-you-need-python-environments-and-how-to-manage-them-with-conda-85f155f4353c)

```
# For example you can create an environment called 'ngs' and install all of the analysis tools there
conda create --name ngs

# you can enter the environment, install specific packages (and specific versions of the packages)
source activate ngs
conda install packagename
conda install -c bioconda packagename

# you can exit the environment
source deactivate ngs

```

## Schedulers
- For most projects I have used an academic HPCC and submit jobs to a queue through a scheduler (slurm/sbatch or pbs/qsub).  To perform the same operation on multiple files I create an array of all filenames in the target directory and submit them as separate, parallel jobs. Each job then can be multi-threaded as per need.
- If you're using a conda environment, then all you need to do is activate the environment before you submit your job to a scheduler. 
- Below I provide an example of how it can be used, for say, performing adapter trimming on multile fastq files using Trim Galore. While this example is for slurm/ sbatch, it works just as well with a pbs/ qsub scheduler (needs appropriate tweaking) and any other ngs tools.

```
source activate ngs

# Usage
sbatch --array=0-10 TrimGalore.sh 

# the --array is a holder for the number of files (numbering starts from 0) on which you want to run the script

```

The actual TrimGalore.sh script looks like this:

```
 #!/bin/bash
#SBATCH --job-name=TrimGalore
#SBATCH --qos=1hr
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --mem=81920
#SBATCH --time=01:00:00
#SBATCH --no-requeue
#SBATCH --mail-type=end,begin,fail
#SBATCH --mail-user=emailID
#SBATCH --output=<full path to files>/slurm-%A_%a.out
#SBATCH --error=<full path to files>/slurm-%A_%a.err

# --job-name: Names the job
# --qos: specifies quality of service and hence job priority
# --nodes: Number of nodes required for a task
# --ntasks: Number of tasks per node
# --cpus-per-task: Number of cpus required per task
# --mem: memory required for the job in mb
# --time: time required in (d-)hh:mm:ss
# --no-requeue: asks not to rerun the script in case of error
# --mail-type: specifies what email updates to send regarding the job
# --mail-user: email address to send the status updates to
# --output: path to put the std out- default name slurm-%j.out (%j=job number)
# --error: path to put the std err file- default name slurm-%j.err (%j=job number)

cd <path to files>

dir= <path to files>

files=(${dir}*.fastq.gz)
# Determine File names 
# single end files are labelled as UniqueID_R#_B#_L#.fastq.gz), where
# UniqueID = some ID, say the genotype
# R# = the biological replicate number 
# B# = the batch or the run number in case samples are in different batches or sequencing runs or flow-cells 
# L# = the Lane number if the samples are run in different lanes within a flow-cell

# use the echo command to see how each element of the array
# echo ${files[0]}
# echo ${files[1]}

file=${files[${SLURM_ARRAY_TASK_ID}]}
# pulls out one element at a time of the array based on $SLURM_ARRAY_TASK_ID (where SLURM_ARRAY_TASK_ID is a special variable for the integers in the array indices).

name=`basename $file .fastq.gz`
# creates variable "name" for the current file based on "file"
# basename is a program (base unix) that strips folder and suffix information from filenames.
# regex to get first part of file name (excludes the extension)
# bname=${name%.*}
# check to make sure that it produces the correct command
# echo "wc -l ${dir}${name} > $bname.out"
# run it!
# wc -l ${dir}${name} > $bname.out

trim_galore ${dir}${name}.fastq.gz -o <full path to target directory>
# Trimgalore does not require a final output file name as it generates the appropriate file name automatically

```
