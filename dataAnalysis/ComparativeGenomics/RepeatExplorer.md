RepeatExplorer2 is a graph-based clustering algorithm that uses unassembled short reads to identify repeat families from low depth genomic sequencing.  Documentation can be found [here](http://repeatexplorer.org/?page_id=818).


```
RepeatExplorer2 clustering is a computational pipeline for unsupervised identification of repeats from unassembled sequence reads. The pipeline uses low-pass whole genome sequence reads and performs graph-based clustering. Resulting clusters, representing all types of repeats, are then examined to identify and classify into repeats groups.

Input data

The analysis requires either single or paired-end reads generated by whole genome shotgun sequencing provided as a single fasta-formatted file. Generally, paired-end reads provide significantly better results than single reads. Reads should be of uniform length (optimal size range is 100-200 nt) and the number of analyzed reads should represent less than 1x genome equivalent (genome coverage of 0.01 - 0.50 x is recommended). Reads should be quality-filtered (recommended filtering : quality score >=10 over 95% of bases and no Ns allowed) and only complete read pairs should be submitted for analysis. When paired reads are used, input data must be interlaced format as fasta file:

example of interlaced input format:

>0001_f
CGTAATATACATACTTGCTAGCTAGTTGGATGCATCCAACTTGCAAGCTAGTTTGATG
>0001_r
GATTTGACGGACACACTAACTAGCTAGTTGCATCTAAGCGGGCACACTAACTAACTAT
>0002_f
ACTCATTTGGACTTAACTTTGATAATAAAAACTTAAAAAGGTTTCTGCACATGAATCG
>0002_r
TATGTTGAAAAATTGAATTTCGGGACGAAACAGCGTCTATCGTCACGACATAGTGCTC
>0003_f
TGACATTTGTGAACGTTAATGTTCAACAAATCTTTCCAATGTCTTTTTATCTTATCAT
>0003_r
TATTGAAATACTGGACACAAATTGGAAATGAAACCTTGTGAGTTATTCAATTTATGTT
```



This tutorial runs sequence reads through the preprocessing steps necessary to run repeatExplorer.

Essentially, we will be quality trimming and trimming adapters from the reads.  Then we select the longest reads that have passed the filter, rename them, and interleave the forward and reverse read pairs.  These reads are then ready for submission to a repeatExplorer web portal or local installation.
Note, it may be time-saving to use a small dataset for the first run.
### Download reads and truncate the file to obtain a subset of reads (no need to process beyond 2 million).
```
module load GIF2/sra-toolkit/2.8.0
#Download and split the fastq
fastq-dump --split-files ERR2377156

#Truncate the file to grap 2million reads. (use zcat if files are gzipped)
head -n 2000000 ERR2377156_1.fastq >2MForward.fastq
head -n 2000000 ERR2377156_2.fastq >2MReverse.fastq
```

### Trim the reads to get rid of low quality and adapters
Any trimmer can be used, just make sure you get the longest reads and trim the correct adapters, as well as only allowing pairs to remain
```
#trim-Galore
trim_galore --length 140 --paired 2MForward.fastq 2MReverse.fastq

#Trimmomatic
java -jar /opt/rit/app/trimmomatic/0.36/bin/trimmomatic-0.36.jar PE -threads 16 -phred33 -trimlog 2M.log 2MForward.fastq 2MReverse.fastq 2MForward.trimmedpaired.fastq 2MForward.trimmedunpaired.fastq 2MReverse.trimmedpaired.fastq 2MReverse.trimmedunpaired.fastq ILLUMINACLIP:/opt/rit/app/trimmomatic/0.36/adapters/TruSeq3-PE.fa:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:140
```

# Convert to fasta and get your subset of reads.
How large of a subset should you use?  The calculation goes like : genome size/read length = # reads needed to get 1X
So for arabidopsis it is 400Mb/(150bp x 2) =1,333,333 paired reads to get 1x.  
So 200k reads = 60MB or 60/400MB = 0.15x coverage
```
awk 'NR % 4 == 1 {print ">" $0 } NR % 4 == 2 {print $0}' 2MForward_val_1.fq |head -n 400000 >200kForwardTrimmedPaired.fasta
awk 'NR % 4 == 1 {print ">" $0 } NR % 4 == 2 {print $0}' 2MReverse_val_2.fq |head -n 400000 >200kReverseTrimmedPaired.fasta

#Here is where we can check to make sure we have the read names appropriately paired between the two files
cat <(grep ">" 200kForwardTrimmedPaired.fasta |awk '{print $1}' ) <(grep ">" 200kReverseTrimmedPaired.fasta |awk '{print $1}' ) |sort|uniq -c |awk '$1==2' |wc
 200000  400000 5512950
#good, 200k reads in each file, with matching names

Now we rename each of the fasta files, because repeatExplorer wants only numbers for read names with a f or a r for read direction.
awk '/^>/{print ">" ++i"_f"; next}{print}' 200kForwardTrimmedPaired.fasta >200kForwardTrimmedPairedRenamed.fasta
awk '/^>/{print ">" ++i"_r"; next}{print}' 200kReverseTrimmedPaired.fasta >200kReverseTrimmedPairedRenamed.fasta
```

The last step before submission is to interleave your fasta reads with their pair.  This is easily done with seqtk
```
seqtk mergepe 200kForwardTrimmedPairedRenamed.fasta 200kReverseTrimmedPairedRenamed.fasta >BothtReadsTrimmedPairedRenamedInterleaved.fasta
```

These reads are now ready to go for the RepeatExplorer program, however there are a number of other options to consider when using RepeatExplorer.
A custom repeat database can be used, when RepeatExplorer is annotating the clustered repeats (i.e. repeatmodeler/repeatmasker).  This may be helpful for later comparisons among datasets.
