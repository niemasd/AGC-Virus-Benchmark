# AGC-Virus-Benchmark
Niema's benchmarking experiment of Assembled Genomes Compressor (AGC) on viral data. I'm using [AGC v3.0](https://github.com/refresh-bio/agc/releases/tag/v3.0) in this experiment.

# SARS-CoV-2 Benchmark
## Getting the Data
In the [supplement](https://oup.silverchair-cdn.com/oup/backfile/Content_public/Journal/bioinformatics/PAP/10.1093_bioinformatics_btad097/1/btad097_supplementary_data.zip?Expires=1680788883&Signature=YqUHPygU3Nq9f95nB2xyklNFcMDX5z5roe6KZ2rtDW~5bK36e7XAjiGTs-b0hwkDyD6OfA-379J~CGCUoycsJB3EctHudsavjCOwMApDO6zVWbHQBRcxUZrGNKJEIiJl3yZ8SKuWheW4WMJ69GHEBr4uGuNUydPtlY8QvZXWXTJ6TbWUoVMd2L8rZk2ilQsPYaWr6ZmYeZIPuuPnChW9uEStlcpHgRRuI6RC2fz7NGA3m6VfLGIcQqVZ78qyGqfjW~BQxkYcU6XqW0cXfQXghOV9EusyBQHrvNoRCiM8R0NEaxyIL1XIMZv2gk~sBKaIFauYFaTP7ITwGQo7b68G-g__&Key-Pair-Id=APKAIE5G5CRDK6RD3PGA) of the [AGC paper](https://doi.org/10.1093/bioinformatics/btad097), section 1.6 points to their COVID dataset:

> Set contains 619,750 complete *SARS-CoV-2* genomes downloaded from the National Center for Biotechnology Information (NCBI) at the end of 2021.
> 
> Data source:
> 
> https://www.ncbi.nlm.nih.gov/datasets/coronavirus/genomes/
> 
> Data availability:
> 
> https://zenodo.org/record/5826274/files/sars-cov-2_ncbi-620k.fa.xz?download=1

I want to be able to benchmark against reference-compressed CRAM, so I want to be able to map against the reference genome using Minimap2, which I don't think(?) supports `xz` compression, so I wanted to `gzip`-compress instead:

```bash
wget -qO- "https://zenodo.org/record/5826274/files/sars-cov-2_ncbi-620k.fa.xz?download=1" | xz --decompress | pigz -9 -p 6 > data/sars-cov-2/sars-cov-2_ncbi-620k.fa.gz
```

## Minimap2 to Samtools to Reference-Compressed CRAM
I then mapped the `gzip`-compressed SARS-CoV-2 multi-genome FASTA to the [NC_045512.2 reference genome](https://www.ncbi.nlm.nih.gov/nuccore/1798174254) using [Minimap2 v2.24-r1122](https://github.com/lh3/minimap2/releases/tag/v2.24), and then piped to [Samtools v1.17](https://github.com/samtools/samtools/releases/tag/1.17) to convert Minimap2's SAM output to reference-compressed CRAM. I also wanted to use GNU `time` to measure runtime and peak memory usage.

```bash
/usr/bin/time -v bash -c "minimap2 -t 4 -a --score-N=0 --secondary=no data/sars-cov-2/reference.fas data/sars-cov-2/sars-cov-2_ncbi-620k.fa.gz | samtools view -@ 4 -C -T data/sars-cov-2/reference.fas --output-fmt-option version=3.1 --output-fmt-option use_lzma=1 --output-fmt-option archive=1 --output-fmt-option level=9 > data/sars-cov-2/sars-cov-2_ncbi-620k.cram" 2> data/sars-cov-2/sars-cov-2_ncbi-620k.cram.log
```

## AGC
I then used AGC to compress the same dataset, also using GNU `time` to measure runtime and peak memory usage.

```bash
/usr/bin/time -v agc create -t 4 -s 10000 -o data/sars-cov-2/sars-cov-2_ncbi-620k.agc data/sars-cov-2/reference.fas data/sars-cov-2/sars-cov-2_ncbi-620k.fa.gz 2> data/sars-cov-2/sars-cov-2_ncbi-620k.agc.log
```

## Results
I ran everything using 4 threads. Note that the CRAM runtime and peak memory include the Minimap2 step.

| Compression/Tool | Size (bytes) | Time (seconds) | Peak Memory (KB) |
| :--------------- | -----------: | -------------: | ---------------: |
| Uncompressed     |   1652191680 |            N/A |              N/A |
| GZ (`pigz -9`)   |          ??? |            ??? |              ??? |
| XZ               |     56312672 |            ??? |              ??? |
| CRAM 3.1         |     26190734 |            ??? |              ??? |
| AGC 3.0          |          ??? |            ??? |              ??? |
