---
title: "Whole Genome Sequencing analysis using bedtools and CNVpytor"
output:
  html_document:
    df_print: paged
  pdf_document: default
---
<br>

### Organise folders & Prep for analysis
* install conda at: https://conda.io/projects/conda/en/latest/user-guide/install/index.html
* open terminal program and run the code below
* `mkdir` creates the folder and subfolder `wgs/` and `/wgs/results/`, (lowercase, spaces, alphabetical characters only)
* `cd` opens *results* as your working folder after saving as `$resultsdir`
* lastly, install conda-based tool, *bedtools*

```{bash}
mkdir -p wgs/results/
$resultsdir="/wgs/results/" 
cd $resultsdir
conda install -c "bioconda/label/cf201901" bedtools
``` 
<br>
Ensure *.bam* files from service provider are sorted by start position and chromosome, and indexed (*.bai* indexed files should appear in the same folder as *.bam* files).  
Ensure *.bed* reference file is sorted in the same way  

* don't forget to replace **PATHtoFile** with your file's location
* sort by using: 
`sort -k1,1 -k2,2n "PATHtoFile/sample1.bam" > "PATHtoFile/sample1_sorted.bam"` or `sort -k1,1 -k2,2n "PATHtoFile/reference.bed" > "PATHtoFile/reference_sorted.bed"`  
*N.B. your reference .bed file is likely in one of your result's mapping or reference folder.*  
  
* save files as variables using:

```{bash}
$sample1_bam="PATHtoFile/sample1_sorted.bam"
$agamp4_bed="PATHtoFile/reference_sorted.bed"
```  
<br>

*optional*
<br>
```{bash}
samtools view $sample1_bam | less -S #view to check .bam file
samtools coverage $sample1_bam #view depth and other stats
```  
<br>

## Analysis of WGS
Create a conda environment and install CNVpytor in it;  
*N.B. cnvpytor command does not overwrite .pytor files, they must first be deleted*

```{bash}
conda create --name cnvpytor_env
conda activate cnvpytor_env
conda config --set channel_priority false
conda install -c bioconda cnvpytor
  # or without Conda use: `git clone https://github.com/abyzovlab/CNVpytor.git`
  # `cd CNVpytor`
  # `pip install .` # for single user (without admin privileges) use: `pip install --user .`
```  
<br>
Initialise CNVpytor project by:  

* saving reference fasta file as a variable and,  
* creating a reference *gc_file*  
*N.B. copy correct pathname/ file location and replace **PATHtoFile** throughout*

```{bash}
$ref_fasta="PATHtoFile/ref_genome.fasta"

cnvpytor -root $resultsdir/AgamP4_gc_file.pytor -gc $ref_fasta -make_gc_file
```

<br>
Using the *gc_file*, either:

* create a separate *ref_genome_conf.py* config file and save with the code below in your results folder (preferred option) or,
* add the code to the **cnvpytor/genome.py** file in cnvpytor's software folder; may be found in a similar location to: */Users/USERNAME/opt/miniconda3/pkgs/cnvpytor-1.2.1-pyhdfd78af_0/site-packages/cnvpytor* - this adds it to the software's genome reference list  
**N.B. below is an example, ensure chromosome names and lengths are changed appropriately for sequences not part of the orignal genome ie transgene**
```{bash}
import_reference_genomes={
  "agamp4_transgene":{
    "name": "AgamP4",
    "species": "Anopheles Gambiae",
    "chromosomes": OrderedDict(
      [("helper_plasmid", (7990, "A")), ("transformation_plasmid", (12339, "A")), ("transgene_name", (8299, "A")),
       ("AgamP4_2L", (49364325, "A")), ("AgamP4_2R", (61545105, "A")),
       ("AgamP4_3L", (41963435, "A")), ("AgamP4_3R", (53200684, "A")), ("AgamP4_UNKN", (42389979, "A")), ("AgamP4_X", (24393108, "S")), ("AgamP4_Y_unplaced", (237045, "S"))]),
    "gc_file":$resultsdir/AgamP4_gc_file.pytor
  }
}
```  
* save new config file as a variable

```{bash}
$ref_genome_conf=$resultsdir/ref_genome_conf.py
```

<br>
Create a *.pytor* file and import config file and read depth (rd) signals from sample's *.bam* file for your specified chromosomes/ sequences
<br>

```{bash}
cnvpytor -conf $ref_genome_conf \
-root $resultsdir/sample1.pytor \
-rd $sample1_bam \
-rg agamp4_transgene \
-chrom helper_plasmid transformation_plasmid transgene_name AgamP4_2L AgamP4_2R AgamP4_3L AgamP4_3R AgamP4_X \
-T $ref_fasta

# check reference genome is correctly detected:
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample1.pytor -ls
```

<br>
Calculate read depth histograms and import alongside stats for each bin; then call cnv regions (try autosomal: 1000 bin size, transgene: 500 bin size for better resolution)

  * recommended minimal bin size depends on sequencing coverage (500 for 30X): https://academic.oup.com/gigascience/article/10/11/giab074/6431715  
  * output of *.tsv* calls includes CNV level, column4: https://github.com/abyzovlab/CNVpytor/blob/master/GettingStarted.md  
```{bash}
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample1.pytor -stat 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample1.pytor -his 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample1.pytor -partition 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample1.pytor -call 500 | > $resultsdir/sample1.calls.500.tsv
```

<br>

### --PLOTS--
* create manhattan plot for autosomal chr.

```{bash}
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample1.pytor -view 10000
```
...then enter the following, row-by-row
```{bash}
set chrom AgamP4_2L AgamP4_2R AgamP4_3L AgamP4_3R AgamP4_X
manhattan
save $resultsdir/sample1_genomeCN_plot.png
quit
```  
  
* create manhattan plot for transgene, backbone etc. (ensure chromosome names match those in your *ref_genome_conf.py* config file)
```{bash}
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample1.pytor -view 500
```
..then enter the following, row-by-row
```{bash}
set chrom helper_plasmid transformation_plasmid transgene_name
manhattan
save $resultsdir/sample1_transgeneCN_plot.png
quit
```  
  
* create rd histogram with different sized bins (*optional: takes some time!*)
```{bash}
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample1.pytor -view 10000
```
...then enter the following, row-by-row
```{bash}
set panels rd
helper_plasmid transformation_plasmid transgene_name AgamP4_2L
save $resultsdir/sample1_rd_plot_10000.png
```  
  


<br>

### ----------------REPEAT for other samples------------------
### Sample #2
```{bash}
$sample2_bam="PATHtoFile/sample2_sorted.bam"

cnvpytor -conf $ref_genome_conf \
-root $resultsdir/sample2.pytor \
-rd $sample2_bam \
-rg agamp4_transgene \
-chrom helper_plasmid transformation_plasmid transgene_name AgamP4_2L AgamP4_2R AgamP4_3L AgamP4_3R AgamP4_X \
-T $ref_fasta

cnvpytor -conf $ref_genome_conf -root $resultsdir/sample2.pytor -stat 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample2.pytor -his 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample2.pytor -partition 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample2.pytor -call 500 | > $resultsdir/sample2.calls.500.tsv
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample2.pytor -ls
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample2.pytor -view 10000

# repeat plots steps
```

### Sample #6
```{bash}
$sample6_bam="PATHtoFile/sample6_sorted.bam"

cnvpytor -conf $ref_genome_conf \
-root $resultsdir/sample6.pytor \
-rd $sample6_bam \
-rg agamp4_transgene \
-chrom helper_plasmid transformation_plasmid transgene_name AgamP4_2L AgamP4_2R AgamP4_3L AgamP4_3R AgamP4_X \
-T $ref_fasta

cnvpytor -conf $ref_genome_conf -root $resultsdir/sample6.pytor -stat 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample6.pytor -his 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample6.pytor -partition 500 10000
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample6.pytor -call 500 | > $resultsdir/sample6.calls.500.tsv
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample6.pytor -ls
cnvpytor -conf $ref_genome_conf -root $resultsdir/sample6.pytor -view 10000

# repeat plots steps
```



--------------------

##### <u> Sources </u>
##### https://github.com/abyzovlab/CNVpytor/blob/master/GettingStarted.md
##### https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8612020/ - CNVpytor: a tool for copy number variation detection
##### https://academic.oup.com/gigascience/article/10/11/giab074/6431715
##### https://bedtools.readthedocs.io/en/latest/content/tools/coverage.html
##### http://statgen.cnag.cat/cnv_analyzer/index.html# - to be used w bedtools
