# Strategy 1: annotation net, curated database, sequence net

Goal: find **all** fungal PX / E3BP proteins (the CBD-bearing subunit of the pyruvate dehydrogenase complex) and separate them from E2 of PDC and from E2 of OGDC/BCKDC.

Why not only one method: annotation misses proteins that have no UniProtKB entry or no detectable domain signature; sequence search from one query is biased toward close matches. Strategy 1 uses the two in turn and lets each correct the other.

## Stages

| # | Stage | What | Notebook | Status |
|---|---|---|---|---|
| 1a | Annotation net, UniProtKB | All fungal proteins with a lipoyl (L), E3-binding (P) or catalytic (C) domain, or a family entry: union of InterPro, Pfam, and Gene3D / SUPERFAMILY / PANTHER signatures | `Family_fetch.ipynb` | built; mock-tested, live fetch in progress |
| 1b | Annotation net, UniParc | The same ID list against UniParc (`feature_id` search) for sequences without a UniProtKB entry (RefSeq-only, WGS, MAG). Keep only entries with no UniProtKB accession | `Family_fetch.ipynb` | **not built**; `feature_id` and taxonomy handling untested |
| 2 | Inspect and curate | Architecture classes (L-P-C, L-P, L-C, ...), one FASTA per architecture, drop inclusions that do not belong, cluster, separate E2 / E3BP / OGDC-BCKDC | `Family_fetch.ipynb` step 5 files, `C_cluster_growth.ipynb` | step 5 files built; curation by hand |
| 3 | Sequence net | Profiles / HMMs from the curated clusters searched against databases annotation cannot reach: `nr` restricted to fungi (includes WGS and MAG translations), optionally tblastn against fungal WGS contigs for genes never predicted | `B_query_to_tree.ipynb`, planned notebook D | not built; manual PSI-BLAST against `nr` first as a test |
| 4 | Feedback | New hits: domain check, add to the database, rebuild profiles, repeat until a round adds nothing | all | not built |

## Design points

- **Merge by sequence, not by accession.** A protein can appear in UniProtKB, RefSeq and a WGS set under different IDs. Merge on the sequence hash (as the reference check does) and keep every source in a `found_by` / `source` column.
- **Source column.** UniProtKB reviewed, UniProtKB unreviewed, UniParc (RefSeq), UniParc (WGS), MAG, `nr`. MAG-derived proteins get a flag: the taxon assignment of a bin can be wrong, so a candidate seen only in one MAG should be checked (neighbouring genes on the contig clearly fungal) before it counts as its own lineage.
- **Names are not evidence.** The yeast gene symbol PDX1 (protein X) clashes with pyridoxal synthase PDX1, so protein X is often named "pyridoxine biosynthesis protein". Classify by domains and sequence. `name_note` flags such entries.
- **Domain evidence is graded.** A Pfam hit is strong; a Gene3D / SUPERFAMILY-only hit (e.g. UPI0044A48F9C, Peltula sp.) is weaker; a diverged E3BP may have no detectable E3-binding hit at all and appears as "lipoyl only". Stage 3 exists for those.
- **UniParc is a sequence archive, not a curated database.** Each distinct sequence appears once, names are whatever the source said, near-identical proteins from many strains stay separate. Clustering removes the redundancy later.
- **Do not widen before checking.** Inspect the UniProtKB union (stage 2 files) and remove unnecessary inclusions before adding UniParc and nr sequences, so errors are not multiplied.
- **Broad entries stay out by default** (`FETCH_BROAD` off): IPR011053 and IPR023213 are generic superfamilies that match unrelated proteins.
- **Deep alignments.** Clustering needs no alignment. For trees, align the L, P and C regions separately (`regions_*.fasta`), or `hmmalign` to a seed HMM, or MAFFT `--retree 2` / `--parttree`, or FAMSA for very large sets.

## Known coverage gaps (what each stage cannot see)

| Stage | Misses |
|---|---|
| 1a | Proteins without a UniProtKB entry (RefSeq-only, WGS-only, MAG); proteins without any matching signature |
| 1b | Proteins in no database UniParc lists; proteins without a matching signature |
| 3 | Genes never predicted (needs tblastn on contigs); sequences too diverged for the profile |

Example of the gap in 1a: UPI001445F4C4 (*Daldinia childiae*, RefSeq XP_033438179, full Pfam PF00364 + PF02817 annotation) is a clear PX but has no UniProtKB entry. UPI0044A48F9C (*Peltula* sp. MAG, WGS KAI9786095) has only Gene3D and SUPERFAMILY matches.

## Open questions for the next design round

1. Does UniParc `feature_id` accept Pfam, signature and InterPro IDs? Does `taxonomy_name`/`organism_id` include descendants of Fungi, or only the exact taxon?
2. How many fungal UniParc sequences without a UniProtKB accession carry these signatures?
3. Sequence net: local `nr` fungal subset (download size) versus remote BLAST versus UniParc only.
4. Cutoffs for adding a hit from stage 3: domain coverage, E-value, profile score margin versus the E2 profiles (the competitive scheme in notebook C).
5. Whether PANTHER subfamily IDs (e.g. PTHR23151:SF82, "PDC protein X component") should be a separate column showing which source each class rests on.

## Files written by Family_fetch (stage 1a / 2)

`proteins.fasta`, `proteins.tsv`, `domains.tsv`, `regions_L/P/C.fasta`, `query_log.tsv`, and `architectures/` (one FASTA per class and architecture, `rare.fasta` for architectures with fewer than `ARCH_MIN_SIZE` proteins, `architectures.tsv` with all architectures).
