# [DOCUMENTATION & SOME SCRIPTS ARE UNDER DEVELOPMENT]

# Contents
 - [Pipeline for generating contact matrix from FASTQ files using HiC-Pro](#Pipeline-for-generating-contact-matrix-from-FASTQ-files-using-HiC-Pro) 
 - [Pipeline for Identification of A/B Compartments](#Pipeline-for-Identification-of-AB-Compartments)
 - [Pipeline for calling TADs](#Pipeline-for-calling-TADs)
 - [Loops and Their Pile-up Analysis Pipeline](#Loops-and-Their-Pile-up-Analysis-Pipeline)


<br>
<br>

# Pipeline for generating contact matrix from FASTQ files using HiC-Pro

## Step 1 - Process Individual Replicates
- Arrange the FASTQ files according to the following directory structure. HiC-Pro considers all readsr within one input folder as one sample
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
  - chromosomes sizes file: Two-column tab-separated text file containing chromosome names and sizes. 
e.g.: https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.chrom.sizes 
   - The bowtie2 indexes: See the bowtie2 manual page for details about how to create such indexes. 
   - (HiC only) A BED file of the restriction fragments after digestion: This file depends on both the restriction enzyme and the reference genome. See the HiC-Pro annotation for details about how to generate this file.
 
- Setup the configuration file: <br>
  Copy and edit the 'config-hicpro.txt' file in your local directory. Modify the options in the configuration file as required. Below are some important options to consider; for more detailed explanations of these options, refer to the HiC-Pro manual. 
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

- Run HiC-pro.
  ```
  Run:
     MY_INSTALL_PATH/bin/HiC-Pro -i FULL_PATH_TO_DATA_FOLDER -o FULL_PATH_TO_OUTPUTS -c MY_LOCAL_CONFIG_FILE
  ```
  
## Step 2 - Merge Replicates
To merge biological replicates after merging technical replicates, we can rerun HiC-Pro starting from the validPairs files generated during the previous analysis. There's no need to start from the FASTQ files again.

- Arrange validPairs files according to the following directory structure.
  ```
  |--PATH_TO_validPairs_FILES
     |--validPairs
        |--validPairs tech_sample1
        |--validPairs tech_sample2
  ```
- Rerun Hic-Pro in stepwise mode.
  ```
  MY_INSTALL_PATH/bin/HiC-Pro -i FULL_PATH_TO_VALIDPAIRS_DATA -o FULL_PATH_TO_OUTPUTS -c MY_LOCAL_CONFIG_FILE -s merge_persample -s build_contact_maps -s ice_norm
  ```
  
## Step 3 - Generate .cool Files
   HiC-Pro outputs contact matrices in .matrix format. However, .cool files are generally accepted by most HiC analysis software and are also easier to handle than other formats.
   Therefore, I convert .matrix files to .cool files using the following 'hicpro2higlass.sh' script that comes with HiC-Pro. 
   ```
   Run:
      HICPRO_PATH/bin/utils/hicpro2higlass.sh -i MATRIX_FILE -r RESOLUTION -c CHROMSIZES_FILE -o OUTPUT_PATH -p NUM_PROC
   ```

## Step 4 - Normalize .cool Files
   Although HiC-Pro can generate normalized .cool files, I prefer to normalize the raw .cool files separately. I use `pipeline_scripts/4_normalize_cool/normalize_cool.py` Python script for normalizing the .cool files. This script uses Iterative correction matrix balancing method from cooler package. The full path to the .cool file should be provided at line number before running.

<br>
<br>

# Pipeline for Identification of AB Compartments 
This pipeline is based on the protocol outlined by Miura et al. in their book1. It utilizes HiCExplorer commands as detailed in the documentation2 and generates A/B compartments starting from a raw .cool matrix. While input for pipeline is a raw .cool matrix, but you can start from a normalized .cool matrix and bypass the step 1 below.

1) Normalize .cool file:
   - (optional) Normalize to equal level of read coverage or value ranges: If you want to compare different Hi-C interaction matrices (samples/replicates), the matrices need to be normalized to equal level of read coverage or value ranges. This can be achieved with ‘hicNormalize’ command as follows. Please refer to hicNormalize documentation for further details.
     ```
     Run:
       hicNormalize -m matrix.cool --normalize smallest -o normalized_matrix.cool
     ```
   - Correct HiC matrix using Iterative Correction (ICE): The Hi-C matrix has to be corrected to remove GC, open chromatin biases and, most importantly, to normalize the number of restriction sites per bin4. However, for this method to work correctly, bins with zero reads assigned to them should be removed as they cannot be corrected. Also, bins with the low number of reads should be removed. Bins with an extremely high number of reads can also be removed from the correction as they may represent copy number variations. To aid in the identification of bins (threshold values) with low and high read coverage, the histogram (diagnostic plot) of the number of reads can be plotted together with the Median Absolute Deviation (MAD) using the ‘hicCorrectMatrix’5 command as shown below.
     ```
     Run:
       hicCorrectMatrix diagnostic_plot –m matrix.cool -o plot_file.png --chromosomes chr
     ```
     Once the thresholds have been decided, the matrix can be corrected as follows.
     ```
     Run:
       hicCorrectMatrix correct -m matrix.cool --correctionMethod ICE –chromosomes chr_list --iterNum 5000 -o corrected.cool --filterThreshold -1.5 5
     ```
  2) Generate observed-over-expected matrix:
     To compute the A/B compartments the matrix needs to be transformed to an observed-over-expected matrix using. ‘hicTransform’7 command.
     ```
     Run:
       hicTransform -m corrected.cool --method obs_exp -o out.cool --chromosomes chr_list
     ```
  3) Calculate Pearson correlation matrix:
     Next, we transform the observed-over-expected matrix into a Pearson correlation matrix by calculating the Pearson correlation values between all pairs of rows and columns as follows. 
     ```
     Run:
       hicTransform -m obs_exp.cool --method pearson -o pearson.cool --chromosomes chr_list 
     ```
  4) Compute first principal component of the Pearson correlation matrix:
     The first principal component (PC) of the Pearson correlation matrix has been shown to correlate with A/B compartments. The first PC track (in bedgraph format) is computed using ‘hicPCA’ command from HiCExplorer as shown below.
     ```
     Run:
       hicPCA -m pearson.cool -o pca1.bedgraph --format bedgraph -we 1 --chromosomes chr_list 
     ```
  5) Calculate gene density:
     Unfortunately, PCA does not provide information about the relationship between the sign of the first principal component (PC) and the A/B compartments. Therefore, it is recommended to use external information, such as histone marks, to identify A/B compartments. One of the straightforward methods to determine A/B compartments is by calculating gene density. The A compartment (active) tends to have a high gene density. It is estimated that regions with high GC content have a higher relative gene density compared to regions with lower GC content. The following Python script calculates the GC content (i.e., gene density) track for given chromosomes. This script requires the TADbit package. Please provide appropriate input arguments at the beginning of the script (lines 9-11).

      ```
      # conda env = tadbit 
  
      import numpy as np 
      import pandas as pd 
      from pytadbit.parsers.genome_parser import get_gc_content, parse_fasta 
      import os.path 
      from subprocess import call 
      
      hic_pro_abs_bed_file=""   # full path to abs.bed file generated by HiC-Pro 
      genome_fasta_file=""      # full path to genome fasta file 
      out_bedgraph_file=""      # full path to output bedgraph file  
          
      df = pd.read_csv(hic_pro_abs_bed_file,sep="\t",header=None) 
      df_chr=df[df[0]==CHR] 
      print(df_chr) 
      
      rich_in_A = get_gc_content(parse_fasta(genome_fasta_file),chromosomes=[CHR],resolution=RESOLUTION, n_cpus=4) 
      print(rich_in_A) 
      print(len(rich_in_A)) 
      df_chr[4]=rich_in_A 
      df_chr=df_chr.dropna(subset=4) 
      df_out=df_chr[[0,1,2,4]] 
      print(df_out) 
      
      df_out.to_csv(out_bedgraph_file, header=False,index=False,sep="\t") 
      ```
    
  6) Plot A/B compartments:
     A/B compartments can be visualized using ‘pyGenomeTracks’13 package. To begin, you need to create a track file that includes the first principal component (PC), gene density, and the contact matrix. Below is an example of how to structure your track file, along with the command to plot the A/B compartments using this file.
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

1) Generate interaction/contact matrix using HiC-Pro:
   Inputs to this custom pipeline are .matrix and _abs.bed files generated by HiC-Pro. You should have them ready before calling TADs. The pipeline to generate interaction/contact matrices using HiC-Pro starting from FASTQ files is outlined in a separate document here.

2) Call TADs:
   The python script to call TADs using TADbit can be downloaded from here. This script outputs TADs in bed files for every given chromosome at specified resolutions. Please modify input arguments at the beginning of the script (lines 13-18) as required and run the script. After a successful run you should get TAD domains bed files in output directory for each chromosome & resolution.

3) Plot TADs:
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

1) Call loops using Mustache:
   Loops can be called using the following command. In the command below, -f is input .cool file, -ch is the list of chromosomes, -r is the resolution, and -o is the output file. If you don't specify the chromosome (-ch) for a .[m]cool  mustache will run on all chromosomes and output loop anchors (coordinates) in a .tsv file specified by -o. For more information on arguments please refer to the Mustache documentation.
   ```
   Run:
       path_to_mustache/mustache/mustache/mustache.py -f cool_file_path -ch chr_list -r res -st 0.7 -pt 0.5 -p 4 -o out.tsv 
   ```
 2) Identify loops whose anchors overlap with histone or TF peaks:
    In this step, we identify loops whose anchors overlap with peaks of particular histone marks or TFs. These selected loops will be used in the next step for pile-up analysis. The following Python script can identify loops whose left anchor overlaps with bed_1 and right anchor overlaps with bed_2. It outputs a .bedpe file containing identified loops. Please modify the input arguments (lines 6-9) as required before running.

    ```
    import numpy as np 
    import pandas as pd 
    import pybedtools 
    
    loops_file_path = "../output/loops_check/loops.tsv"                   # path to loops.tsv 
    bed_1_file_path = "../input/bed_files/DE_Ctrl_PRDM1.0.7rpm.bed"       # Left histone/TF mark 
    bed_2_file_path = "../input/bed_files/DE_Ctrl_H2AK119Ub1.all.bed"     # Right histone/TF mark 
    out_file_path = "../output/check.bedpe"                               # path to output file 

    # intersection 
    loops_df=pd.read_csv(loops_file_path, sep="\t") 
    loops_df["loop_number"]=np.arange(loops_df.shape[0]) 

    loops_left_df=loops_df[["BIN1_CHR","BIN1_START","BIN1_END","loop_number"]]  
    loops_right_df=loops_df[["BIN2_CHROMOSOME","BIN2_START","BIN2_END","loop_number"]] 

    loops_left_bed=pybedtools.BedTool.from_dataframe(loops_left_df) 
    loops_right_bed=pybedtools.BedTool.from_dataframe(loops_right_df) 

    bed_1 = pybedtools.BedTool(bed_1_file_path) 
    bed_2 = pybedtools.BedTool(bed_2_file_path) 

    loops_left_bed_1_intersect=loops_left_bed.intersect(bed_1, u=True).to_dataframe()
    loops_left_bed_1_intersect=loops_left_bed_1_intersect.rename(columns={"chrom":"BIN1_CHR", "start":"BIN1_START", "end":"BIN1_END", "name":"loop_number"}) 
    loops_right_bed_2_intersect=loops_right_bed.intersect(bed_2, u=True).to_dataframe() 
    loops_right_bed_2_intersect=loops_right_bed_2_intersect.rename(columns={"chrom":"BIN2_CHROMOSOME", "start":"BIN2_START", "end":"BIN2_END", "name":"loop_number"}) 

    loops_bed_1_on_left=loops_df.merge(loops_left_bed_1_intersect, how="inner", on=["BIN1_CHR", "BIN1_START", "BIN1_END", "loop_number"]) 
    loops_bed_2_on_right = loops_df.merge(loops_right_bed_2_intersect, how="inner", on=["BIN2_CHROMOSOME", "BIN2_START", "BIN2_END", "loop_number"]) 
    loops_bed_1_on_left_bed_2_on_right = loops_bed_1_on_left.merge(loops_bed_2_on_right, how="inner") 
    loops_bed_1_on_left_bed_2_on_right.to_csv(out_file_path, sep="\t", header=None, index=False) 
    ```

3) Aggregate (pile-up) analysis:
   Aggregate analysis is carried out using coolpuppy package4. The script to perform pile-up analysis can be downloaded from here. It takes the loop file (in bedpe format) generated in the previous step and performs off-diagonal pile-up analysis using coolpuppy, producing a pile-up analysis plot. Please modify the input arguments (lines 13-18) at the beginning of the script as needed and run the script. After a successful run, you should obtain a plot of the pile-up analysis.
