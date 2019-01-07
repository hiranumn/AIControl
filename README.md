# AIControl.jl

[![Build Status](https://travis-ci.org/hiranumn/AIControl.jl.svg?branch=master)](https://travis-ci.org/hiranumn/AIControl.jl)


AIControl makes ChIP-seq assays **easier**, **cheaper**, and **more accurate** by imputing background data from mass control data available in public.

Here is an overview of AIControl framework from our paper. 
![alt text](images/concept.png)

*Figure 1: (a) Comparison of AIControl to other peak calling algorithms. (left) AIControl
learns appropriate combinations of publicly available control ChIP-seq datasets to impute background
noise distributions at a fine scale. (right) Other peak calling algorithms use only one
control dataset, so they must use a broader region (typically within 5,000-10,000 bps) to estimate
background distributions. (bottom) The learned fine scale Poisson (background) distributions are
then used to identify binding activities across the genome. (b) An overview of the AIControl
approach. A single control dataset may not capture all sources of background noise. AIControl
more rigorously removes background ChIP-seq noise by using a large number of publicly available
control ChIP-seq datasets*

## Major Updates
- (12/14/2018) Cleared all deprecations. AIControl now works with Julia 1.0. Please delete the precompiled cache from the previous versions of AIControl. You may do so by deleting the `.julia` folder. 
- (12/15/2018) Updated some error messages to better direct users (12/13/2018).
- (1/7/2019) Made AIControl Pkg3 compatible for Julia 1.0.3

## Installation 
AIControl can be used on any **Linux** or **macOS** machine. While we tested and validated on **Windows** machines, we believe that it is easier for you to set the AIControl pipeline up on the Unix based systems. Running AIControl on a `.fastq` file to get a `.narrowPeak` file requires the following sets of programs and packages installed. We will explain how to install them in sections below.
- Julia (Julia 0.7 and above)
- bowtie2 
- samtools
- bedtools

## Installing Julia
```
cd
wget https://julialang-s3.julialang.org/bin/linux/x64/1.0/julia-1.0.3-linux-x86_64.tar.gz
tar xvzf julia-1.0.3-linux-x86_64.tar.gz
echo  'export PATH=$PATH:~/julia-1.0.3/bin' >> ~/.bashrc
source .bashrc
```

## Installing utility softwares
AIControl expects a sorted `.bam` file as an input and outputs a `.narrowpeak` file. Typically, for a brand new ChIP-seq experiment, you would start with a `.fastq` file, and you will need some external softwares for converting the `.fastq` file to a sorted `.bam` file. Here, we provide a list of such external softwares. The recommended way of installing these softwares is to use package management systems, such as `conda`. Please download anaconda Python distribution from [here](https://anaconda.org/anaconda/python). Install `anaconda` and run the following commands.
- **bowtie2**: ` conda install -c bioconda bowtie2 ` for aligning a `.fastq` file to the hg38 genome
- **samtools**: ` conda install -c bioconda samtools ` for sorting an alinged bam file
- **bedtools**: ` conda install -c bioconda bedtools ` for converting a bam file back to a fastq file (OPTIONAL for Step 3.1)

## Julia modules required for AIControl

AIControl module is coded in **Julia 1.0**. You can download Julia from [here](https://julialang.org/).  

Before you start, make sure your have the following required packages installed. The easiest way to do this is to open `julia` and start typing in following commands. 
- `using Pkg`
- `Pkg.add("JLD2")`
- `Pkg.add("FileIO")`
- `Pkg.clone("https://github.com/hiranumn/AIControl.jl.git")`

## Data files required for AIControl
AIControl uses a massive amount of public control data for ChIP-seq (roughly 450 chip-seq runs). We have done our best to compress them so that you only need to download about **4.6GB** (can be smaller with the `--reduced` option). These files require approximately **13GB** of free disk space to unfold. You can unfold them to anywhere you want as long as you specify the location with the `--ctrlfolder` option. **The default location is `./data`.** **[Here](https://drive.google.com/open?id=1Xh6Fjah1LoRMmbaJA7_FzxYcbqmpNUPZ) is a link to a Google Drive folder that contains all compressed data.** Please download the following two data files. Use `tar xvjf file.tar.bz2` to untar. 
- `forward.data100.nodup.tar.bz2` (2.3GB):   
- `reverse.data100.nodup.tar.bz2` (2.3GB):  

We also have other versions of compressed control data, where duplicates are not removed (indicated with `.dup`, and used with the `--dup` option) or the number of controls are subsampled. Please see the **OtherControlData** folder. 

## Paper
We have an accompanying paper in BioRxiv evaluating and comparing the performance of AIControl to other peak callers in various metrics and settings. **AIControl: Replacing matched control experiments with machine learning improves ChIP-seq peak identification** ([BioRxiv](https://www.biorxiv.org/content/early/2018/03/08/278762?rss=1)). You can find the supplementary data files and peaks files generated by the competing peak callers on [Google Drive](https://drive.google.com/open?id=1Xh6Fjah1LoRMmbaJA7_FzxYcbqmpNUPZ).

## How to use AIControl (step by step)

**Step 1: Map your FASTQ file from ChIP-seq to the `hg38` assembly from the UCSC database.**  
We have validated our pipeline with `bowtie2`. You can download the genome assembly data from [the UCSC repository](http://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz). In case you need the exact reference database that we used for bowtie2, they are available through our [Google Drive](https://drive.google.com/open?id=1Xh6Fjah1LoRMmbaJA7_FzxYcbqmpNUPZ) as a zip file named `bowtie2ref.zip`.  

*Example command:*  
`bowtie2 -x bowtie2ref/hg38 -q -p 10 -U example.fastq -S example.sam`  

Unlike other peak callers, the core idea of AIControl is to leverage all available control datasets. This requires all data (your target and public control datasets) to be mapped to the exact same reference genome. Our control datasets are currently mapped to the hg38 assembly from [the UCSC repository]. **So please make sure that your data is also mapped to the same assembly**. Otherwise, our pipeline will report an error.
   
**Step 2: Convert the resulting sam file into a bam format.**  

*Example command:*  
`samtools view -Sb example.sam > example.bam`  
   
**Step 3: Sort the bam file in lexicographical order.**  
If you go through step 1 with the UCSC hg38 assembly, sorting with `samtools sort` will do its job.  

*Example command:*  
`samtools sort -o example.bam.sorted example.bam`  

**Step 3.1: If AIControl reports an error for a mismatch of genome assembly**  
You are likely here, because the AIControl script raised an error. The error is most likely caused by a mismatch of genome assembly that your dataset and control datasets are mapped to. Our control datasets are mapped to the hg38 from [the UCSC repository](http://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz). On the other hand, your bam file is probably mapped to a slightly differet version of the hg38 assembly or different ordering of chromosomes (a.k.a. non-lexicographic). For instance, if you download a bam file directly from the ENCODE website, it is mapped to a slightly different chromosome ordering of hg38. A recommended way of resolving this issue is to extract a fastq file from your bam file, go back to step 1, and remap it with bowtie2 using the UCSC hg38 assembly. `bedtools` provides a way to generate a `.fastq` file from your `.bam` file.  
 
*Example command:*  
`bedtools bamtofastq  -i example.bam -fq example.fastq`  

We will regularly update the control data when a new major version of the genome becomes available; however, covering for all versions with small changes to the existing version is not realistic.
   
**Step 4: Download data files and locate them in the right places.**  
As stated, AIControl requires you to download precomputed data files. Please download and extract them to the `./data` folder, or otherwise specify the location with `--ctrlfolder` option. Make sure to untar the files.    

**Step 5: Run AIControl as julia script.**  
You are almost there. If you clone this repo, you will find a julia script `aicontrolScript.jl` that uses AIControl functions to identifiy locations of peaks. Here is a sample command you can use.  

`julia aicontrolScript.jl example.bam.sorted --ctrlfolder=/scratch/hiranumn/data --name=test`

Do `julia aicontrolScript.jl --help` or `-h` for help.

We currently support the following flags. 

- `--dup`: using duplicate reads \[default:false\]
- `--reduced`: using subsampled control datasets \[default:false\]
- `--xtxfolder=[path]`: path to a folder with xtx.jld2 (cloned with this repo) \[default:./data\]
- `--ctrlfolder=[path]`: path to a control folder \[default:./data\]
- `--name=[string]`: prefix for output files \[default:bamfile_prefix\]
- `--p=[float]`: pvalue threshold \[default:0.15\]

## Simple trouble shooting
Make sure that:
- You are using Julia 1.0.
- You downloaded necessary files for `--reduced` or `--dup` if you are running with those flags.
- You sorted the input bam files according to the UCSC hg38 assembly as specified in Step 1 (and 3.1).

## We have tested our implementation on:
- macOS Sierra (2.5GHz Intel Core i5 & 8GB RAM)
- Ubuntu 18.04 
- Windows 8.0

If you have any question, please e-mail to hiranumn at cs dot washington dot edu.
