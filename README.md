# [DOCUMENTATION & SOME SCRIPTS ARE UNDER DEVELOPMENT]

# Contents
 - [Pipeline for generating contact matrix from FASTQ files using HiC-Pro](#Pipeline-for-generating-contact-matrix-from-FASTQ-files-using-HiC-Pro) 
 - [Pipeline for Identification of A/B Compartments](#Pipeline-for-Identification-of-AB-Compartments)
 - [Pipeline for calling TADs](#Pipeline-for-calling-TADs)
 - [Loops and Their Pile-up Analysis Pipeline](#Loops-and-Their-Pile-up-Analysis-Pipeline)


<br>
<br>

# Pipeline for generating contact matrix from FASTQ files using HiC-Pro

```
Scripts: 
- /pipeline_scripts/1_hic_pro
- /pipeline_scripts/2_hicpro_to_cool_mcool
- /pipeline_scripts/4_normalize_cool
```
## Usage
### Step 1 - Process Individual Replicates
- Arrange the FASTQ files according to the following directory structure. HiC-Pro considers all reads within one input folder as one sample.
  ```
  |--PATH_TO_FASTQ_FILES
     |--sample1
        |--file1_R1.fastq.gz
        |--file1_R2.fastq.gz
        |--file2_R1.fastq.gz
        |--file2_R2.fastq.gz
        |-- ...
     |--sample2
        |-- file1_R1.fastq.gz
        |-- file1_R2.fastq.gz
  ```
- Generate annotation files:
  - **chromosomes sizes file:** Two-column tab-separated text file containing chromosome names and sizes. 
e.g.: https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.chrom.sizes 
  - **The bowtie2 indexes:** See the bowtie2 manual page for details about how to create such indexes. 
  - **(HiC only) A BED file of the restriction fragments after digestion:** This file depends on both the restriction enzyme and the reference genome. See the HiC-Pro annotation for details about how to generate this file.
 
- Setup the configuration file: <br>
  Copy and edit the `pipeline_scripts/1_hic_pro/config-hicpro.txt` file in your local directory. Modify the options in the configuration file as required. Below are some of the important options to consider; for more detailed explanations of these options, refer to the HiC-Pro manual. 
  - PAIR1_EXT = Keyword for first mate detection. Default: _R1 
  - PAIR2_EXT = Keyword for second mate detection. Default: _R2 
  - REFERENCE_GENOME = Reference genome prefix used for genome indexes. Default: hg19 
  - GENOME_SIZE = Full path to chromosome size file. 
  - MIN_CIS_DIST4 = [Important option for Micro-C data] Filter short range contact below the specified distance. Example: 1000 
  - BIN_SIZE = Resolution of contact maps to generate (space separated). Default: 20000 40000 150000 500000 1000000 
  - (Hi-C only) GENOME_FRAGMENT = Full path to BED file with restriction fragments. 
  - (Hi-C only) LIGATION_SITE = Ligation site sequence(s) used for reads trimming.      - Depends on the fill in strategy. Note that multiple ligation sites can be specified (comma separated). Example: AAGCTAGCTT 
  - (Hi-C only) MIN_FRAG_SIZE = Maximum size of restriction fragments to consider for the Hi-C processing. Example: 100 
  - (Hi-C only) MAX_FRAG_SIZE = Maximum size of restriction fragments to consider for the Hi-C processing. Example: 100000 
  - (Hi-C only) MIN_INSERT_SIZE = Minimum sequenced insert size. Shorter 3C products are discarded. Example: 100 
  - (Hi-C only) MAX_INSERT_SIZE = Maximum sequenced insert size. Larger 3C products are discarded. Example: 600

- Run HiC-pro. <br>
  Modify the parameters in `pipeline_scripts/1_hic_pro/run_hicpro_fastq.sh` according to following command:
  ```
  MY_INSTALL_PATH/bin/HiC-Pro -i FULL_PATH_TO_DATA_FOLDER -o FULL_PATH_TO_OUTPUTS -c MY_LOCAL_CONFIG_FILE
  ```
  Then,
  ```
  Run:
      ./run_hicpro_fastq.sh
  ```
  
### Step 2 - Merge Replicates
To merge replicates after processing individual replicates, we can rerun HiC-Pro starting from the validPairs files generated during the individual replicates processing. There's no need to start from the FASTQ files again.

- Arrange validPairs files according to the following directory structure.
  ```
  |--PATH_TO_validPairs_FILES
     |--validPairs_dir
        |--validPairs tech_sample1
        |--validPairs tech_sample2
  ```
- Rerun Hic-Pro in stepwise mode. <br>
  Modify the parameters in `pipeline_scripts/1_hic_pro/run_hicpro_merge_validPairs.sh` according to following command:
  ```
  MY_INSTALL_PATH/bin/HiC-Pro -i FULL_PATH_TO_VALIDPAIRS_DATA -o FULL_PATH_TO_OUTPUTS -c MY_LOCAL_CONFIG_FILE -s merge_persample -s build_contact_maps -s ice_norm
  ```
  Then,
  ```
  Run:
      ./run_hicpro_merge_validPairs.sh
  ```
  
### Step 3 - Generate .cool Files
   HiC-Pro outputs contact matrices in .matrix format. However, .cool files are generally accepted by most HiC analysis software and are also easier to handle than other formats.
   Therefore, I convert .matrix files to .cool files using the following 'hicpro2higlass.sh' script that comes with the HiC-Pro.
   
   Modify the parameters in `pipeline_scripts/2_hicpro_to_cool_mcool/1_create_cool_from_hic_pro_matrix.sh` according to following command:
   ```
   HICPRO_PATH/bin/utils/hicpro2higlass.sh -i MATRIX_FILE -r RESOLUTION -c CHROMSIZES_FILE -o OUTPUT_PATH -p NUM_PROC
   ```
   Then,
   ```
   Run:
       ./1_create_cool_from_hic_pro_matrix.sh
   ```

### Step 4 - Normalize .cool Files
   Although HiC-Pro can generate normalized .cool files, I prefer to normalize the raw .cool files separately. I use `pipeline_scripts/4_normalize_cool/normalize_cool.sh` script for normalizing the .cool files. This script uses Iterative correction matrix balancing method from cooler package.

Modify the following parameters in `pipeline_scripts/4_normalize_cool/normalize_cool.sh`:
```
# Path to .cool file
in_file_path="../output/cool_matrix/hPSC_merged_bio_rep/1000/hPSC_merged_bio_rep_1000_random_chr_removed_unnormalized.cool"

# Resolution of .cool file
in_resolution=1000

# Output directory path
out_path="../output/cool_matrix/hPSC_merged_bio_rep/1000"

# Modify output file name here
out_file_path="${out_path}/hPSC_merged_bio_rep_1000_random_chr_removed_normalized.cool"
```
Then,
```
Run:
    ./normalize_cool.sh
```

<br>
<br>

# Pipeline for Identification of AB Compartments 
This pipeline is based on the protocol outlined by Miura et al. in their book. It utilizes HiCExplorer to generates A/B compartments starting from a raw .cool matrix.

```
Scripts: 
- /pipeline_scripts/6_ab_comparmtments
```

## Usage

### Step 1 - Calculate 1st PC
Modify the following parameter in `pca.sh`:
```
# Path to input .cool file
cool_file_path=""

# Output directory path
out_path=""

# Chromosome name
CHR=""
```
```
Run:
    ./pca.sh
```

### Step 2 - Plot A/B compartments:
A/B compartments can be visualized using pyGenomeTracks package. To begin, you need to create a track file that includes the first principal component (PC), gene density, and the contact matrix. Below is an example of how to structure your track file, along with the command to plot the A/B compartments using this file.

Example track.ini file:
     
```
[x-axis] 
where = top 
 
[spacer] 
 
[pca1 bedgraph] 
file = pca1_Seq021_MicroC_merged_rep_chr1_500000.bedgraph 
color = blue 
height = 5 
transform = no 
negative_color = red 
height = 2 
title = PCA 1 

[rich_in_A.bedgraph] 
file = rich_in_A_Seq021_MicroC_merged_rep_chr1_500000.bedgraph 
color = orange 
type = line:2 
height = 2 
title = GC content 
 
[spacer] 
 
[hic matrix] 
file = obs_exp_Seq021_MicroC_merged_rep_chr1_500000.cool 
depth =  250_000_000 
file_type = hic_matrix_square 
colormap = coolwarm 
transform = log 
min_value = -2 
max_value = 2 
show_masked_bins = true 
title = log(O/E)
```
```
Run:
   pyGenomeTracks --tracks tracks.ini --region CHR:star-end --outFileName out.png 
```
     
<br>
<br>

# Pipeline for calling TADs
This a custom pipeline for identification TAD boundaries using TADbit1 tad caller. The output of different TAD callers does not always agree with each other and can vary substantially depending on the algorithm used.

```
Scripts:
- /pipeline_scripts/7_tads
```

## Usage 

### Step 1 - Generate interaction/contact matrix using HiC-Pro:
Inputs to this custom pipeline are .matrix and _abs.bed files generated by HiC-Pro. You should have them ready before calling TADs. The pipeline to generate interaction/contact matrices using HiC-Pro starting from FASTQ files is outlined above.



### Step 2 - Call TADs:
Use `call_tads_tadbit.py` script to call TADs. This script uses TADbit package & outputs TADs in bed files for every given chromosome at specified resolutions. Please modify input arguments at the beginning of the scriptm as required and run the script. After a successful run you should get TAD domains bed files in output directory for each chromosome & resolution.

Modify the following parameters in `call_tads_tadbit.py`:
```
sample_name="DE"         # name of the sample
in_path=""               # full path to raw directory inside HiC-Pro output directory 
out_path=""              # path to output dir to tad bed files
chr_sizes_file_path=     # path to genome sizes file
chr_list=[]              # list of chromosomes
resolution_list=[]       # list of resolutions (should be present in the HiC-Pro output directory
```
Then,
```
Run:
    python call_tads_tadbit.py
```

### Step 3 - Plot TADs:
TADs can be visualized using ‘pyGenomeTracks’2 package. To begin, you need to create a track file that includes contact matrix and TAD domains. Below is an example of how to structure your track file, along with the command to plot the TADs using this file.

```
[x-axis] 
where = top 

[spacer] 

[hic matrix] 
file = matrix.cool 
depth = 50000000 
file_type = hic_matrix
colormap = coolwarm 
transform = no 
max_value = 1 
min_value = -1 
title = hFG-lib548+lib549_merged-AB matrix 

[tads] 
file = domains.bed 
display = triangles 
border_color = black 
color = none 
overlay_previous = share-y 
line_width = 2.5 
```

```
Run:
    pyGenomeTracks --tracks tracks.ini --region CHR:star-end --outFileName out.png
```

<br>
<br>

# Loops and Their Pile-up Analysis Pipeline
The intended analysis of this pipeline is the pile-up (aggregate) analysis of the loops whose anchors overlap with peaks of particular histone marks or TF1,2.  The input for pipeline is a normalized .cool matrix file.

```
Scripts:
- /pipeline_scripts/8_loops_and_pileup_analysis/1_call_loops
- /pipeline_scripts/8_loops_and_pileup_analysis/6_loops_and_histone_marks_or_tfs_bed_intersection
- /pipeline_scripts/8_loops_and_pileup_analysis/2_pileup_analysis_aggregate_analysis/off_diagonal_pileup_analysis
```

## Usage

### Step 1 - Call loops using Mustache:
Loops can be called using the Mustache script - `1_call_loops/call_loops_mustache.sh`. Modify the parameters in the script as described below. 

- -f is input .cool file
- -ch is the list of chromosomes. If you don't specify the chromosome (-ch) for a .[m]cool  mustache will run on all chromosomes
- -r is the resolution, 
- -o is the output file.
For more information on arguments please refer to the Mustache documentation.

Then,
```
Run:
    ./call_loops_mustache.sh
```

### Step 2 - Identify loops whose anchors overlap with histone or TF peaks:
In this step, we identify loops whose anchors overlap with peaks of particular histone marks or TFs. These selected loops will be used in the next step for pile-up analysis. This `6_loops_and_histone_marks_or_tfs_bed_intersection/2_loops_histone_marks_intersection.py` Python script can identify loops whose left anchor overlaps with bed_1 and right anchor overlaps with bed_2. It outputs a .bedpe file containing identified loops. Please modify the followng input arguments in the python script required before running.

```
loops_file_path = "../output/loops_check/loops.tsv"                   # path to loops.tsv file
bed_1_file_path = "../input/bed_files/DE_Ctrl_PRDM1.0.7rpm.bed"       # Left histone mark/TF
bed_2_file_path = "../input/bed_files/DE_Ctrl_H2AK119Ub1.all.bed"     # Right histone mark/TF
out_file_path = "../output/check.bedpe"                               # path to output file
```
Then,
```
Run:
    ./2_loops_histone_marks_intersection.sh
```

### Step 3 - Aggregate (pile-up) analysis:
Aggregate analysis is carried out using coolpuppy package. The script to perform pile-up analysis is present at `/pipeline_scripts/8_loops_and_pileup_analysis/2_pileup_analysis_aggregate_analysis/off_diagonal_pileup_analysis/pileup_analysis_off_diagonal.py`. It takes the loop file (in bedpe format) generated in the previous step and performs off-diagonal pile-up analysis using coolpuppy, producing a pile-up analysis plot. Please modify the following input arguments at the beginning of the script as needed and run the script. After a successful run, you should obtain a plot of the pile-up analysis.

```
cool_file_path="../output/DE_merged_bio_rep_1000_normalized.cool"        # path to normalized .cool file
bedpe_file_path="../output/merge_loops/loops_merged.bedpe"               # path to loops .bedpe file
flank = 25_000                                                           # flanking distance around loops
out_file_path="../output/check.h5"                                       # path to output file
out_figure_path = "../output/check.png"                                  # path of ouput pile-up plot
SAMPLE="DE"                                                              # name of the sample
```
Then,
```
Run:
    ./pileup_analysis_off_diagonal.sh
```
