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
## Run fastStrucutre for K=1 to K=8 (or more if needed)
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
# Graph K values using R
## Load necessary libraries 
```bash
library(tidyverse)
library(RColorBrewer)  # for custom color palettes
```
## Load Q file and check it is correct
```bash
Q <- read.table("fs_geno75.6.meanQ")
colnames(Q) <- paste0("Pop", 1:ncol(Q))
```
## Read in sampleID file
```bash
sample_ids <- read.table("bamlist_nodupes.txt", header = FALSE, stringsAsFactors = FALSE)
colnames(sample_ids) <- "SampleID"
```
## Assign sample IDs to Q
```bash
Q$SampleID <- sample_ids$SampleID
head(Q)
```
## Read popmap
```bash
popmap <- read.table("pop_map_nodupes_final.txt", header = TRUE, stringsAsFactors = FALSE)
```
## Merge city info into Q
```bash
Q <- Q %>%
  left_join(popmap[, c("sample", "city", "year")],
            by = c("SampleID" = "sample"))
```
## Sort by city and year
```bash
Q <- Q %>% arrange(city, year)
Q$Sample <- 1:nrow(Q)  # numeric index for plotting
```
## Convert to long format
```bash
Q_long <- Q %>%
  pivot_longer(cols = starts_with("Pop"),
               names_to = "Population",
               values_to = "Ancestry")
```
## Make graph pretty 
```bash
#set city colors and boundaries
K <- length(grep("Pop", colnames(Q)))
pop_colors <- brewer.pal(K, "Set2")

# Vertical lines between cities
city_breaks <- Q %>%
  group_by(city) %>%
  summarize(end = max(Sample))

# Midpoints for city labels
city_labels <- Q %>%
  group_by(city) %>%
  summarize(mid = mean(Sample))
```
## Plot fastStructure bars sorted by city
```bash
ggplot(Q_long, aes(x = Sample, y = Ancestry, fill = Population)) +
  geom_bar(stat = "identity") +
  scale_fill_manual(values = pop_colors) +
  #Solid black lines that stop at bar height
  geom_segment(data = city_breaks,
               aes(x = end + 0.5,
                   xend = end + 0.5,
                   y = 0,
                   yend = 1),
               inherit.aes = FALSE,
               color = "black",
               linewidth = 0.6) +
  #City labels under bars
  geom_text(data = city_labels, aes(x = mid, y = -0.03, label = city),
            inherit.aes = FALSE, angle = 45, hjust = 1, vjust = 1, size = 3) +
  #Flip y-limits to give space for city labels
  scale_y_continuous(expand = expansion(mult = c(0.2, 0.1))) +
  theme_minimal() +
  labs(x = "Individuals (grouped by city)", y = "Ancestry proportion", fill = "Cluster") +
  theme(axis.text.x = element_blank(),
        axis.ticks.x = element_blank(),
        plot.margin = margin(6, 6, 20, 6))  # extra bottom margin for labels
```

