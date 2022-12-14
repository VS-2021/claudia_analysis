# README for claudia_analysis :dna:
This repository contains step-by-step instructions on how to analyze genotyping-by-sequencing (GBS) data using Claudia's samples and Reneth's Itasca-C12 GWAS population.

## Contents
[Helpful tips to get you started](#Helpful-tips-to-get-you-started)<br>
[Directory setup](#Directory-setup)<br>
[Adapter trimming](#Adapter-trimming)<br>
[Read alignment](#Read-alignment)<br>
[Index BAM files](#Index-BAM-files)<br>
[SNP calling](#SNP-calling)<br>
[Filter SNP calls](#Filter-SNP-calls)<br>
[Principal component analysis](#Principal-component-analysis)<br>
[Create SNP matrix](#Create-SNP-matrix)

## Helpful tips to get you started
* In order to submit a bash script (ending in `.sh`), you need to use `sbatch` followed by the script name. For example: `sbatch run_cutadapt.sh`
* You can check the status of jobs you have running using `squeue`. For example (using your username): `squeue -u caste007`
* If you need to cancel a job for whatever reason, use `scancel` followed by the JOBID that corresponds to the job you want to cancel. You can see this when you run `squeue`
* Pay attention to paths. They are specific to my directory and yours will be different.
* Make sure you are working in the global scratch drive (`/scratch.global`) to avoid using up our group's allocation which is 5 Tb as of December 2021.
* You will need to make your own directory within `/scratch.global` to keep things clean; `cd` to that directory, then use `mkdir` to make a directroy called `caste007`. You should carry out these scripts in that directory or within a project-specific directory within `caste007`.
* Files are removed from the global scratch drive after 30 days, so don't leave important files there. Move important files to your own computer or the Google drive.
* Files that you want to keep permanently (important `VCF` files or figure files can move moved to the shared directory. We are trying to structure the shared directory so that it is easy to understand and find files.  For example, files from GBS analysis from 2021 should go here:` /home/jkimball/shared/sequencing/GBS/2021`. Right now, there is only one directory called `november_2021_gbs` which contains output from the work that I did for the complete set.
* You can check how much of our storage we are using by typing: `groupquota -g jkimball`
* You might notice that many of the files or directories contain a string of numbers at the beginning of the name. This represents a date in YY-MM-DD format and is a standard practice in scientific computing. The benefit is that it automatically sorts your files or directories by date.

## Directory setup
The (raw) data are located here:
```bash
/home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008
```

In order to make the file with the full paths to the data (`FASTQ` files), I decided to take an iterative approach to specify which samples to include based on my knowledge of what the sample names are.
```bash
ls /home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008/5-most*fastq.gz >> claudia_analysis_filenames.txt
```
**Note:** Two of the `FASTQ` files (one forward _R1_ and one reverse _R2_) have an underscore between the "5" and "most" in the sample name. The following line of code accounts for this difference.
```bash
ls /home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008/5_most*fastq.gz >> claudia_analysis_filenames.txt
```

The next group of samples are either "All R" or "All S".
```bash
ls /home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008/All*fastq.gz >> claudia_analysis_filenames.txt
```
The remainder of the samples begin with either "NK" or "RS" depending on which farm the samples were collected from.
```bash
ls /home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008/NK*fastq.gz >> claudia_analysis_filenames.txt
ls /home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008/RS*fastq.gz >> claudia_analysis_filenames.txt
```
At this point, you can check to make sure all of the files we expect to be there are actually there:
```bash
cat claudia_analysis_filenames.txt | wc -l
```
The result is 84. Since we have 42 samples and each sample has 2 files (one forward read and one reverse read), this matches our expectations.

Since these are Itasca-C12 samples, it might also be helpful to have data from Reneth's GWAS project.
```bash
ls /home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008/IC12*fastq.gz >> claudia_analysis_filenames.txt
```
If you count the files again, you should get 400. We have 200 samples in total (there are 158 Itasca-C12 samples), so everything is in order.

The next step is to move into the `R` statistical environment to go from file names to a workable CSV file that will be used in the next step of the directory structure setup.
```R
# Read in data using the data.table package
library(data.table)
fread("claudia_analysis_filenames.txt", header=F) -> x

# Change the column name from V1 to something more informative (filename)
setnames(x, "filename")
# Add a new column called sample_number. It will initially contain the entire filename, but we will work to retain only the sample number
x[, sample_number := filename]
# Strip off first part of filename until sample number begins (S) but do not include it.
x[, sample_number := sub("^.+[S]", "", sample_number)]
# Strip off end of the filename (after the sample number) ... begins with "_R1" or "_R2"
x[, sample_number := sub("_[R1].+$", "", sample_number)]

# Convert sample numbers to numerical and add leading zeros to all samples (to help with sorting).
x[, sample_number := sprintf("%04d", as.numeric(sample_number))]

# Reorder rows in ascending order
x[order(sample_number)] -> x

# Set column order (my personal preference for sample_number to come first)
setcolorder(x, c("sample_number", "filename")) -> x

# Write output to CSV
write.csv(x, file="211226_claudia_analysis_sample_names_and_numbers.csv", row.names=FALSE, col.names=FALSE, sep=",", quote=FALSE)

# Save table as an R object
save(x, file="211226_claudia_analysis_sample_names_and_numbers.Rdata")
```
After that is done, use the `CSV` file using `bash` to create the directory structure.<br>
**Note:** The `echo $i` part is not really necessary. I just included it to watch the progress.
```bash
cat 211226_claudia_analysis_sample_names_and_numbers.csv | cut -f 1 -d , \
	| while read i; do
	d=Sample_$i
	echo $i
	mkdir -p $d
	done
```
Once that is done, you will probably notice that there is a directory called `Sample_sample_number` which is an artefact of the code. I probably could change the code so that the header isn't interpreted as a sample name, but it's also super easy to just remove it after the code finishes. You can easily remove it with a one-liner:
```bash
rm -rf Sample_sample_number
```

Next, you should make a file with the list of directories. This `txt` file will come in handy for future steps of the GBS analysis.
```bash
ls Sample*/ -d | tr -d / > 211226_claudia_analysis_sample_directory_list.txt
```
This next step is necessary because we are working with paired-end reads. We are doing it because the file `211222_reneth_gwas_sample_names_and_numbers.csv` contains 2 lines per sample (one for the forward read and one for the reverse read).
```bash
awk 'FNR%2' 211226_claudia_analysis_sample_names_and_numbers.csv > 211226_claudia_analysis_file_list_every_other.csv
```
_Make sure you open the resulting file using_ `vi` _to manually remove the header line_. Once that is done, we can make symbolic links (symlinks) to point to the data rather than take up disk space by needlessly duplicating the original files. **Note** that when I analyzed the original dataset, I set `n` to start at 73 and then increased that value by 1 with each iteration of the `while` loop. Since this iteration of the analysis only contains the GWAS samples, there are gaps in the sequence of sample numbers, necessitating a different approach. The approach I used involves extracting the sample number (`Snumber`) from the file name and using that rather than relying on counting iterations through the loop.
```bash
# Make symlinks to GBS data
cat 211226_claudia_analysis_file_list_every_other.csv | cut -f 9 -d / \
	| while read i; do
	STEM=$(echo $i | rev | cut -f 3,4,5,6,7,8 -d "_" | rev)
	Snumber=$(echo $i | rev | cut -f 3 -d "_"| rev | sed 's/^S//g')
	n=$(printf "%04d\n" $Snumber)
	echo $STEM
	ln -s /home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008/${STEM}_R1_001.fastq.gz Sample_$n/Sample_${n}_R1.fq.gz
	ln -s /home/jkimball/data_delivery/umgc/2021-q4/211108_A00223_0697_BHNY3NDSX2/Kimball_Project_008/${STEM}_R2_001.fastq.gz Sample_$n/Sample_${n}_R2.fq.gz
	done
```
In the next step, we will move back to the `R` statistical environment to create a sample key.
```R
# Move back to R
library(data.table)

# Read in data
x <- fread("211226_claudia_analysis_sample_names_and_numbers.csv")

# Change column names
setnames(x, c("sample_number", "sample_name"))

# Add leading zeros
x[, sample_number := sprintf("%04d", as.numeric(sample_number))]
# Add "Sample_" to each sample number
x[, sample_number := paste0("Sample_", sample_number)]

# Remove beginning the beginning part of the filename to remove the part of the path that is no longer necessary to keep
x[, sample_name := sub("^.+Project_008/", "", sample_name)]

# Remove trailing part of filenames (sample names)---ultimately, we only need one line per sample, not two (a consequence of having 2 files per sample for paired-end reads)
x[, sample_name := sub("_[R1].+$", "", sample_name)]
x[, sample_name := sub("_[R2].+$", "", sample_name)]

# Retain unique values only
x <- unique(x)

# Save to CSV
write.csv(x, file="211226_claudia_analysis_sample_key.csv", row.names = FALSE, sep=",", quote=FALSE)
```

This next bit is just to check if the symlinks are all correct. You can do it manually (by going into each directory and typing `ls -lh`, paying special attention to the first and last to make sure the sample numbers match the "S number" in the filenames provided by UMGC. This way of checking it just quickly puts the same information into a single file so you can view them all at once. **Note:** I only did the forward reads (_R1_) because if they are correct, the reverse reads (_R2_) will also be correct.
```bash
for i in $(cat 211226_claudia_analysis_sample_directory_list.txt);
do
ls -lh ${i}/${i}_R1.fq.gz >> check_symlinks_full_paths.txt
done
```
## Adapter trimming
The next step in the process is to trim the adapters. Since this is my second time processing this dataset, there is no reason to run the FastQC quality reports.

The script to submit for the adapter trimming is [run_cutadapt.sh](adapter_trimming/run_cutadapt.sh) which depends on/calls the script [cutadapt_wrapper_script.sh](adapter_trimming/cutadapt_wrapper_script.sh). That means they need to be in the same directory in order to work properly.

## Read alignment
After you have trimmed the adapters from the reads, the next step is to align the reads to the genome. We use the Burrows-Wheeler Aligner Maximal Exact Match (BWA-MEM). Use [run_bwa.sh](alignment/run_bwa.sh) for this step. It will take several hours, so it is best to let this run overnight. 200 samples took about 14 hours.

After the alignmet step has completed, you can use the following one-liner to make a file containing the relative paths to each `BAM` file. This will be helpful in the coming steps.
```bash
ls */*sorted.bam > claudia_sorted_bam_files.txt
```
You can also check the file sizes for the `BAM` files with `ls -lh */*sorted.bam`. Most of the files should be in the hundreds of megabytes range (100M-400M). Some might be less (70M, for example). But if you see something exceptionall low (92 or 24K), something has gone wrong and you will want to redo the alignment _for those samples only_).

## Index `BAM` files
Before doing the SNP calling itself, you need to index the `BAM` files using the script [index_bams.sh](index_bams/index_bams.sh). It uses the file [claudia_sorted_bam_files.txt](helper_files/claudia_sorted_bam_files.txt) that you made in the previous step, so make sure the script can find that file. For 200 samples, this will take about 35 minutes.

## SNP calling
Now, we proceed to the actual SNP-calling step. Use the script [scythe_mpileup.sh](snp_calling/scythe_mpileup.sh) to do this. Like the alignment step, this will take several hours. One parameter to pay particular attention to is the `-q 20` option. This means that _the minimum mapping quality (MQ) for an alignment to be used_ is 20. This is a measurement of the quality of the read being mapped, not the base at the SNP. You can increase the stringency up to `-q 60`, although `-q 20` is acceptable It's a phred-like score and means that the minimum acceptable probability for a read being correct is 99%. Reads with a lower mapping quality are filtered out. Many (if not most) reads will have an even higher probability of being right.**Note:** This script uses [GNU Parallel](https://www.gnu.org/software/parallel/), so make sure you cite the program in any manuscript that uses results from these data. You can get the citation info by running `parallel --citation`. (You'll need to run `module load parallel` first.)

## Filter SNP calls
Once the SNP calling step is done, you will have a list of 2,183 g-zipped `VCF` files (`.vcf.gz`). There is one file per chromosome/scaffold. Most of these don't contain any SNPs at all, so it isn't worth looking at them. They're also quite small (insignificant) in terms of length of genome sequence. Since we renamed the scaffolds, you no longer need to worry about the original scaffold names deliered to us by Dovetail. You will need to make a file like [vcf_file_list.txt](helper_files/vcf_file_list.txt). I made this manually because it's just a list of the files we actually want to look at (instead of all 2,183). The first one is `211227_snp_calling_results_ZPchr0001.vcf.gz`, the second is called `211227_snp_calling_results_ZPchr0002.vcf.gz`, and so forth all the way through `211227_snp_calling_results_ZPchr0016.vcf.gz`. However, the 17th scaffold (which is important because it is greater than 4 Mb in size contains the Northern Wild Rice _sh4_ ortholog) is called `211227_snp_calling_results_ZPchr0458.vcf.gz`. It was originally Scaffold_453, but we didn't include it in the renaming process because it wasn't among the 15 largest scaffolds.  If we had included it, it would have been ZPchr0017.

Anyway, use the script [filter_with_vcftools.sh](filter_vcfs/filter_with_vcftools.sh) to filter the `VCF` files in order to meet your desired parameters. The way the script is currently written, the parameters are:<br>
* Maximum 10% missing data (`--max-missing 0.90`). _I know this is confusing, but it's correct._
* Bi-allelic sites only (`--min-alleles 2 --max-alleles 2`)
* Minor allele frequency is 0.03 (`--maf 0.03`)
* No indels (`--remove-indels`)
* Minimum depth of 8 reads required at a SNP (`--minDP 8`)

**Note:** So far, most of the software programs we have been using so far have already been installed by the Minnesota Supercomputing Institute (MSI). That's why you can use them by calling `module load` and then referring to them in your code simply by calling the name of the program (e.g., `bwa`, `samtools`, or `bcftools`). [`VCFtools`](https://vcftools.github.io/index.html) is different because I had to install it myself and refer to the place where it is installed in my script (`~/vcftools/bin/vcftools`) rather than just using `vcftools`.

Once the `VCF` files have been filtered according to your desired parameters, you can move on to the next step: putting the SNP calls into a `CSV`-formatted SNP matrix. However, I also like working with [plink](https://zzz.bwh.harvard.edu/plink/index.shtml), especially for performing principal component analysis (PCA). As a first step in that analysis, I merge the 17 filtered `VCF` files into a single merged `VCF` file with [concat_filtered_vcfs.sh](filter_vcfs/concat_filtered_vcfs.sh).

## Principal component analysis
The first step in the pricipal component analysis (PCA) is to run the script [run_plink.sh](plink/run_plink.sh) which will convert the merged `VCF` file into plink format and generate the _eigenvalue_ and _eigenvector_ files that are necessary to make the PCA plots in the R statistical environment. The _eigenvalue_ tells you how much variation is explained by each of the principal components while the _eigenvector_ provides plotting coordinates for each sample on the PCA plot(s). The next step is to run the script [run_plot_plink_pca.sh](plink/run_plot_plink_pca.sh) which requires the R script [plot_plink_pca.R](plink/plot_plink_pca.R) and the `CSV` file [ItascaC12_Claudia_sample_key.csv](helper_files/ItascaC12_Claudia_sample_key.csv). The path to that `CSV` file is hard-coded into my script so you'll need to change that to reflect where you put the file. You will notice several components to the bash script:<br>
```bash
Rscript plot_plink_pca.R  claudia_analysis_pca.eigenvec claudia_analysis_pca.eigenval 211227_claudia_analysis.pdf 211227_claudia_analysis.Rdata
```
The first part tells bash to use R; the next position (technically the "0" position is the name of the R script you want to use). The following files (positions 1, 2, 3, and 4 or `args[1]`, `args[2]`, `args[3]`, and `args[4]` in the R language) are file names that are inserted into the R script in lieu of hard-coding them into the script itself. This way, you can repeatedly use the same R script.

One of the output files will be a `PDF` file with multiple PCA plots (through the first 8 PCs). We use `PDF` format because it is superior in terms of maintaining resolution. However, you can export to other file types including `PNG`, `JPG`, or `TIF` for presentations, publications, or to use in your own GitHub repositories like the example below (generated in the process of creating this README document).<br>
<img src="images/claudia_analysis_PC1_vs_PC2.png" width="500">

**Note:** Other than the key file, the other components are hard-coded into the script because they are specific to your project are: 1) plot title, 2) colors that correspond to sample types ("Resistant", "Susceptible", and "Itasca-C12"), and 3) the plot legend.

## Create SNP matrix
The conversion of the `VCF` files to a single tab-separated value (`TSV`) file containing the SNP data is done using `AWK`. The script that does the work is called [normalize.awk](create_snp_matrix/normalize.awk) and is launched using [run_normalize_awk.sh](create_snp_matrix/run_normalize_awk.sh). The `TSV` file is in long format. That means it has one line per SNP + Individual (sample) combination. You'll want to put this into a SNP matrix (`CSV` format) that has one column per sample and one row per SNP. The R script [filter_snps_and_make_wide_format.R](create_snp_matrix/filter_snps_and_make_wide_format.R) will do this. You run the R script using the bash script [run_filter_snps_and_make_wide_format.sh](create_snp_matrix/run_filter_snps_and_make_wide_format.sh). It takes the `TSV` format as input (`args[1]` in R) and outputs a `CSV` file containing the SNP matrix and an `.Rdata` file from the process (`args[2]` ad `args[3]`, respectively).

**Note:** When I initially wrote [filter_snps_and_make_wide_format.R](create_snp_matrix/filter_snps_and_make_wide_format.R), I wrote it so that low-confidence heterozygotes were removed. I did this because SAMtools will call a heterozygote with only a single alternate read to support it. Ideally, you would like to have a little more confidence that it's a true heterozygote and not a sequencing or alignment error. I initially used a minimum depth of 6 (`--minDP 6` in VCFtools) and therefore wanted to use a minimum of 3 alternate alleles (3/6 or 50% of reads at a given SNP); however, rather than removing the SNP entirely, it is just replaced with an "NA" value. I think this is because there are homozygous SNP calls in other individuals at the same locus. So, it becomes a matter of which is worse: some heterozygotes that you might be unsure of or an increase in the amount of missing data. You can change this by removing the `#` symbol in front of `#y[GT==0 | GT == 2 | (GT==1 & DV >= 3)] -> z` if you want to opt for missing data instead.

Congats! You should now have the SNP data you need to do for down-stream analyses (GWAS, pop-gen, etc).:tada:
