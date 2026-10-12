# homology_clusters

Colab/Jupyter notebooks for finding remote homologs of a protein region and sorting them into competing clusters (for example E2 versus E3BP of the pyruvate dehydrogenase complex).

| Notebook | Purpose |
|---|---|
| `A_build_databases.ipynb` | Build standalone MMseqs2 databases with taxonomy (a taxonomic subset of UniRef, or from a FASTA). Run once. |
| `B_query_to_tree.ipynb` | One query sequence: search, filter, cluster, align, tree. Optional stage 2 (class assignment in a second database). |
| `C_cluster_growth.ipynb` | Several anchor sequences: controlled competitive cluster growth by iterated profile searches; contested sequences are held out. |

All three are Colab forms (code cells collapsed) and also run in plain Jupyter. Each step has a text cell saying what it does, what to expect and which parameters matter. Results are checkpointed.

## Status
Tested end to end only on small mock databases on Linux. Not yet run on real UniRef-scale data. Notebook B still contains Linux-only shell calls; A and C avoid them.

## Setup on a Linux machine or cluster
```bash
conda create -n seqwork -c conda-forge -c bioconda python=3.11 \
    mmseqs2 mafft fasttree iqtree biopython pandas numpy matplotlib jupyterlab
conda activate seqwork
git clone https://github.com/bforsbe/homology_clusters
cd homology_clusters
jupyter lab        # or jupyter notebook
```
Run A first (it needs internet access and, for UniRef50, about 10 GB of disk for the download), then point `DATABASE` in B or C at the folder it made. Windows: use WSL2; the setup steps are at the top of notebook A.

## License
GPL-3.0, see `LICENSE`.
