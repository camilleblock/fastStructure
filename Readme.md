# fastStructure 
<img width="2824" height="1359" alt="image" src="https://github.com/user-attachments/assets/ac41bfd6-7725-4ec6-aec6-b9fdf6e1358c" />

## Introduction
This page is a work in progress!
This repo explains how to run fastStructure on Linux and R from a .vcf file. This pipeline runs in Linux and R and relies on Plink (version PLINK/2.00a3.7-gfbf-2023a) and fastStructure (version fastStructure/1.0-foss-2023a-Python-2.7.18). For this entire pipeline, it may be necessary to split each step up into smaller sample groups. All code should be run in scratch.

Contact: Camille Block (camilleblock@vt.edu)

## Remember to create a SLURM Script
```bash
#!/bin/bash
#SBATCH --nodes=2
#SBATCH --cpus-per-task=20
#SBATCH --time=12:00:00
#SBATCH --job-name STACKS
#SBATCH --account=bedbug
#SBATCH --partition=normal_q
```
## Load programs 
```bash
module load EasyBuild/5.1.2
module load PLINK/2.00a3.7-gfbf-2023a
```
## EasyBuild fastStructure if necessary
```bash
module use $HOME/.local/easybuild/modules/all
module load fastStructure/1.0-foss-2023a-Python-2.7.18
eb /apps/arch/software/EasyBuild/5.1.2/easybuild/easyconfigs/f/fastStructure/fastStructure-1.0-foss-2023a-Python-2.7.18.eb --robot
```
## Convert vcf file to bed file
```bash
plink --vcf snps_merged.integer.maf01.geno5.final.vcf.gz \
       --make-bed \
       --out snps_merged.integer.maf01.geno5.final
```
## Run fastStrucutre for K=1 to K=8 (or more if needed 
```bash
structure.py -K1 \
  --input snps_merged.integer.maf001.geno75.final \
  --output fs_geno75
```
## Choose best K value based on model complexity and model components
Model complexity that maximizes marginal likelihood = the K with the highest model evidence.
Model components used to explain structure in data = the smallest K that adequately explains the population structure (often the value reported in the fastStructure paper)
```bash
chooseK.py --input=fs_geno75
```
