# NGS-QC-Common-Questions

_1. When evaluting the left read of paired-end reads (R1), the right read (or R2) often has lower quality than the left read from Illumina sequencing. Why and how to mitigate this?_
Ans: It is very common for Illumina R2 reads to have lower quality than R1. This happens because the chemistry degrades over the longer run time, and the "phasing" (asynchrony of the cluster) accumulates by the time the second read starts.This difference in quality is just an artifact of the paired-end Illumina sequencing. Here are some recommendations to remediate this:
- Run FastQC/MultiQC: Confirm if the quality drop is a steady decline toward the end of the read or if it’s a sudden crash.
- Adapter & Quality Trimming: Use a tool like Trimmomatic or Cutadapt. You can set a "sliding window" (e.g., SLIDINGWINDOW:4:20) to crop the ends of R2 where the quality dips below a certain threshold.
- Use "Soft-Clipping" during Alignment: Most modern aligners (like BWA-MEM or Bowtie2) will automatically "soft-clip" low-quality ends that don't match the reference, effectively ignoring the "bad" parts of R2 without losing the "good" parts.
- Read Merging: If your library inserts are short enough that R1 and R2 overlap, use a tool like PEAR or Flash. These tools use the high-quality R1 sequence to "correct" the overlapping low-quality portion of R2.
- Check for "Dark Cycles": If R2 is significantly worse from the very first base, it could be a hardware or reagent issue (like a bubble in the flow cell or a failed re-synthesis step).

2. If we discard the low‑quality R2 reads and end up with unequal read counts between R1 and R2, can we still proceed to the alignment step?
Ans: Most paired-end aligners (like BWA-MEM, Bowtie2, or STAR) require that the R1 and R2 files have the exact same number of reads and that the reads are in the same relative order. If you have more reads in R1 than R2, the aligner will eventually encounter a name mismatch or reach the end of one file while the other still has data, causing the run to crash or produce corrupted results. If you trimmed or filtered your R1 and R2 files separately, a low-quality read might be discarded from R2 while its corresponding mate in R1 is kept because it passed the quality threshold. This creates "orphan" or "singleton" reads that throw the files out of sync.
There are two main options to resolve this:
- Re-run Trimming in "Paired-End Mode":
Use a tool like Trimmomatic, fastp, or Cutadapt and provide both R1 and R2 at the same time. These tools are "pair-aware"; if one read in a pair fails the quality filter, they will either discard the entire pair or move the surviving mate to a separate "unpaired" or "singleton" file, keeping your main R1 and R2 files perfectly synced.
- Repair the Files:
If you no longer have the raw data and need to fix your current mismatched files, use a tool specifically designed to re-sync them:
BBMap's repair.sh: This is the gold-standard tool for this problem. It compares the read IDs in both files and outputs matched R1/R2 files, placing any orphans into a third file.
Custom Scripts: Community scripts like fastqCombinePairedEnd.py can also be used to match IDs
