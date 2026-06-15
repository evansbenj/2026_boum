# Using pygm to inform boum and vice versa

I have pygm WGS data here:
```
/home/ben/projects/rrg-ben/ben/2025_allo_PacBio_assembly/Adam_boum_genome_assembly/pygm_8samples_WGS
```
Plan:
Make fem and mal specific kmer dbs; subtract mal from fem; pull out reads with fem-specific kmerz, assemble, map to boum

# make a female kmer db
```
#!/bin/sh
#SBATCH --job-name=makemeryldb
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH --time=48:00:00
#SBATCH --mem=128gb
#SBATCH --output=makemeryldb.%J.out
#SBATCH --error=makemeryldb.%J.err
#SBATCH --account=rrg-ben

/home/ben/projects/rrg-ben/ben/2025_bin/meryl/build/bin/meryl count fem_pygm_*_trim_R[1,2].fq.gz threads=4 memory=128 k=29 output pygmfem_meryldb.out
```
# make a male kmer db
```
#!/bin/sh
#SBATCH --job-name=makemeryldb
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH --time=48:00:00
#SBATCH --mem=128gb
#SBATCH --output=makemeryldb.%J.out
#SBATCH --error=makemeryldb.%J.err
#SBATCH --account=rrg-ben

/home/ben/projects/rrg-ben/ben/2025_bin/meryl/build/bin/meryl count mal_pygm_*_trim_R[1,2].fq.gz threads=4 memory=128 k=29 output pygmmal_meryldb.out

```
# make a female-specific kmer db by using the difference flag
```
#!/bin/sh
#SBATCH --job-name=meryl_intersect
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH --time=2:00:00
#SBATCH --mem=128gb
#SBATCH --output=meryl_intersect.%J.out
#SBATCH --error=meryl_intersect.%J.err
#SBATCH --account=rrg-ben

# sbatch 2026_meryl_intersect.sh fq_meryl kmermeryl
/home/ben/projects/rrg-ben/ben/2025_bin/meryl/build/bin/meryl difference ${1} ${2} output in_${1}_but_not_${2}.meryl
```


# extract reads with fem-specific kmerz:
(keep in mind that these are not "real" paired end reads - so subsequent assembly needs to be done with single ends concatenated reads
```
#!/bin/sh
#SBATCH --job-name=makemeryldb
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=4
#SBATCH --time=2:00:00
#SBATCH --mem=128gb
#SBATCH --output=makemeryldb.%J.out
#SBATCH --error=makemeryldb.%J.err
#SBATCH --account=rrg-ben



for r1 in fem_pygm_*_trim_R1.fq.gz; do
    # Derive the R2 file name from R1
    r2="${r1/*_trim_R1.fq.gz/*_trim_R2.fq.gz}"
    
    # Extract the base sample name for output files
    prefix=$(basename "$r1" _trim_R1.fq.gz)
    
    echo "Processing sample: $prefix"
    /home/ben/projects/rrg-ben/ben/2025_bin/meryl/build/bin/meryl-lookup -include \
	 -sequence "$r1" \
         -mers in_pygmfem_meryldb.out_but_not_pygmmal_meryldb.out.meryl \
         | pigz -c > "${prefix}_femspecific_R1.fastq.gz"

/home/ben/projects/rrg-ben/ben/2025_bin/meryl/build/bin/meryl-lookup -include \
         -sequence "$r2" \
         -mers in_pygmfem_meryldb.out_but_not_pygmmal_meryldb.out.meryl \
         | pigz -c > "${prefix}_femspecific_R2.fastq.gz"

done
```

# Assemble with Spades

# Identify CDS using XL and filtered blast:
```
blastn -query /home/ben/projects/rrg-ben/ben/2025_allo_PacBio_assembly/Adam_boum_genome_assembly/XL_CDS_only_nospaces.fasta -db contigs.fasta_blastable -outfmt "6 qseqid sseqid length qlen" | awk '($3 / $4) >= 0.80' > XL_CDS_to_pygm_femspecific.txt
```
