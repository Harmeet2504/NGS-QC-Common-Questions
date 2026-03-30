# A Compilation of Common Questions in NGS Data Analysis

__1. When evaluting the left read of paired-end reads (R1), the right read (or R2) often has lower quality than the left read from Illumina sequencing. Why and how to mitigate this?__<br> 
__Ans:__ It is very common for Illumina R2 reads to have lower quality than R1. This happens because the chemistry degrades over the longer run time, and the "phasing" (asynchrony of the cluster) accumulates by the time the second read starts.This difference in quality is just an artifact of the paired-end Illumina sequencing. Here are some recommendations to remediate this:
- Run FastQC/MultiQC: Confirm if the quality drop is a steady decline toward the end of the read or if it’s a sudden crash.
- Adapter & Quality Trimming: Use a tool like Trimmomatic or Cutadapt. You can set a "sliding window" (e.g., SLIDINGWINDOW:4:20) to crop the ends of R2 where the quality dips below a certain threshold.
- Use "Soft-Clipping" during Alignment: Most modern aligners (like BWA-MEM or Bowtie2) will automatically "soft-clip" low-quality ends that don't match the reference, effectively ignoring the "bad" parts of R2 without losing the "good" parts.
- Read Merging: If your library inserts are short enough that R1 and R2 overlap, use a tool like PEAR or Flash. These tools use the high-quality R1 sequence to "correct" the overlapping low-quality portion of R2.
- Check for "Dark Cycles": If R2 is significantly worse from the very first base, it could be a hardware or reagent issue (like a bubble in the flow cell or a failed re-synthesis step).

__2. If we discard the low‑quality R2 reads and end up with unequal read counts between R1 and R2, can we still proceed to the alignment step?__<br> 
__Ans:__ Most paired-end aligners (like BWA-MEM, Bowtie2, or STAR) require that the R1 and R2 files have the exact same number of reads and that the reads are in the same relative order. If you have more reads in R1 than R2, the aligner will eventually encounter a name mismatch or reach the end of one file while the other still has data, causing the run to crash or produce corrupted results. If you trimmed or filtered your R1 and R2 files separately, a low-quality read might be discarded from R2 while its corresponding mate in R1 is kept because it passed the quality threshold. This creates "orphan" or "singleton" reads that throw the files out of sync.
There are two main options to resolve this:
- Re-run Trimming in "Paired-End Mode":
Use a tool like Trimmomatic, fastp, or Cutadapt and provide both R1 and R2 at the same time. These tools are "pair-aware"; if one read in a pair fails the quality filter, they will either discard the entire pair or move the surviving mate to a separate "unpaired" or "singleton" file, keeping your main R1 and R2 files perfectly synced.
- Repair the Files:
If you no longer have the raw data and need to fix your current mismatched files, use a tool specifically designed to re-sync them:
BBMap's repair.sh: This is the gold-standard tool for this problem. It compares the read IDs in both files and outputs matched R1/R2 files, placing any orphans into a third file.
Custom Scripts: Community scripts like fastqCombinePairedEnd.py can also be used to match IDs

__3.What information is encoded in the bitwise flag found in the second column of a SAM file, and how can it be decoded to understand read properties?__  <br>
__Ans:__ Bitwise flags in SAM files (column 2) are compact numerical codes representing boolean properties of a sequencing read, such as mapping status, strand, and pairing, used to save disk space. A single integer (e.g., 16, 99, 163) is the sum of various binary flags, which can be decoded to understand the read's alignment characteristics. Examples: A flag of 16 (0x10) indicates the read is on the reverse strand. A flag of 1040 is a combination of 1024 (duplicate) and 16 (reverse strand). The Broad Institue provides a [utility tool](https://broadinstitute.github.io/picard/explain-flags.html) to decode SAM flags.

__4.What are read groups? Which analyses use read groups? What is its advantage?__ <br> 
__Ans:__In Next-Generation Sequencing (NGS), read groups are metadata tags (often starting with @RG in a SAM/BAM file) that identify the specific origin of a set of sequencing reads. Think of them as a "digital passport" for your data. A read group typically includes several key identifiers:
ID: A unique string (like a flowcell ID and lane number).
SM (Sample): The name of the biological sample.
LB (Library): The specific DNA library preparation.
PL (Platform): The sequencing technology used (e.g., ILLUMINA).
PU (Platform Unit): Usually the flowcell/lane/barcode combination. 
Which analyses use them?
Many downstream tools rely on read groups to function correctly, most notably: 
GATK Best Practices: Tools like Base Quality Score Recalibration (BQSR) and MarkDuplicates use read group info to distinguish between biological variation and technical artifacts.
Variant Calling: Callers (like HaplotypeCaller or DeepVariant) use the SM tag to group reads by sample, even if those reads came from multiple different sequencing runs or lanes.
Multi-Sample Merging: If you sequence the same sample across three lanes, read groups allow the software to treat them as a single biological entity while still tracking lane-specific biases. 
What is the advantage?
Technical Error Correction: By knowing which reads came from the same flowcell or lane, software can identify and fix systematic errors (like a "bad" batch or a specific laser glitch) without affecting the rest of your data.
Batch Effect Detection: It makes it easy to spot if one specific library prep or sequencing run is behaving differently than others.
Traceability: If you find an issue with a specific sample later, you can trace it back to the exact machine, lane, and date it was sequenced.
Efficiency: You can merge BAM files from different runs into a single file while maintaining the ability to differentiate the sources for statistical modeling-

While most modern pipelines (like GATK or Snakemake-based workflows) are designed to handle read groups automatically, you should still perform a manual "sanity check" at the beginning and end of the alignment step.
The software is "garbage in, garbage out"—if you provide the wrong metadata, the software will proceed blindly, leading to flawed results.
What one should inspect:
The Header (Before/During): Run samtools view -H your_file.bam | grep '@RG' to ensure the @RG tags are actually there. If they are missing, many downstream tools (like GATK) will simply crash.
The SM Tag: Ensure the SM (Sample) tag is identical for all files belonging to the same biological sample. If you have two different library preps for "SampleA" but name one SM:Sample_A and the other SM:SampleA, the caller will treat them as two different people.
The LB Tag: Ensure different library preparations have different LB tags. This tells the software that duplicates found across these files are likely optical/sequencing duplicates, not PCR duplicates from the same prep.
Example Scenario: The "Zombie" Variant (False Positive SNV)
Imagine you are calling variants and find a highly suspicious mutation that only appears in one out of your three biological replicates.
The Detection: You use IGV (Integrative Genomics Viewer) to look at the alignment. You see the "variant" is present in about 20% of the reads at that position.
The Manual Inspection: You color the reads by Read Group (right-click in IGV > Color alignments by > read group).
The Discovery: You notice that 100% of the reads containing the mutation belong to Read Group A (Lane 1), while Read Group B (Lane 2) and Read Group C (Lane 3) show 0% mutation.
The Conclusion: Because the mutation is perfectly correlated with a specific sequencing lane rather than being distributed across all lanes, it is likely a technical artifact (e.g., a bubble in the flowcell or a calibration error in that specific lane) rather than a real biological variant.


Reference:
1. [FASTQC manual](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/)
2. [FastQC detailed explanation](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/Help/3%20Analysis%20Modules/)
3. [HBC Training](https://hbctraining.github.io/Intro-to-variant-analysis/lessons/02_fastqc.html)
