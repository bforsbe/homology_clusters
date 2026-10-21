# homology_clusters

Colab/Jupyter notebooks for finding remote homologs of a protein region and sorting them into competing clusters (for example E2 versus E3BP of the pyruvate dehydrogenase complex).

| Notebook | Purpose |
|---|---|
| `A_build_databases.ipynb` | Build standalone MMseqs2 databases with taxonomy (a taxonomic subset of UniRef, or from a FASTA). Run once. |
| `B_query_to_tree.ipynb` | One query sequence: search, filter, cluster, align, tree. Optional stage 2 (class assignment in a second database). |
| `Family_fetch.ipynb` | Collect all UniProt proteins carrying lipoyl / E3-binding / catalytic domain entries (InterPro and Pfam IDs, union over IDs and taxa), label domain architectures, write FASTA and region files, optionally build a small database for B and C. Needs internet. |
| `Family_fetch_strategy1.ipynb` | Strategy 1, stage 1a: the same fetch limited to a net of E3-binding (PF02817, IPR004167, IPR036625) and E2 / protein X family (IPR045257) entries; lipoyl and catalytic entries are annotate-only (they add domain positions, never proteins). Also searches UniParc for sequences without a UniProtKB entry and writes `cbd_candidates.fasta`. See `docs/strategy_1.md`. |
| `C_cluster_growth.ipynb` | Several anchor sequences: controlled competitive cluster growth by iterated profile searches; contested sequences are held out. |

All three are Colab forms (code cells collapsed) and also run in plain Jupyter. Each step has a text cell saying what it does, what to expect and which parameters matter. Results are checkpointed.

## Status
Tested end to end only on small mock databases on Linux. Not yet run on real UniRef-scale data. Notebook B still contains Linux-only shell calls; A, C and Family_fetch avoid them. Family_fetch was tested only against a local mock of the InterPro API, not the live service.

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

## Strategy

The overall plan (annotation net, curated database, sequence net) is in [docs/strategy_1.md](docs/strategy_1.md).

For contributors (human or Claude Code): [docs/handoff.md](docs/handoff.md) records the aims, priorities and known traps.
