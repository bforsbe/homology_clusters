# Handoff for Claude Code: aims, priorities and context that the code does not show

Read this first, then `docs/strategy_1.md` (the staged plan) and `README.md` (notebooks). This file records intent, decisions and cautions from the design conversation. Statements marked **Stated by the user** are the user's own aims; anything marked **Suggestion** is Claude's proposal, not yet agreed.

## 1. What this project is for

**Stated by the user:** find *all* fungal PX / E3BP homologs, with the aim of finding "EVERYTHING", and separate three groups that share a core architecture: E2 of the pyruvate dehydrogenase complex (PDC), E2 of the 2-oxoglutarate / branched-chain complexes (OGDC, BCKDC), and the E3-binding protein (E3BP, "protein X", PX). The user works on fungal PX / E3BP and is likely the author of papers on it.

**Stated by the user, most recent and overriding:** the search should be as wide as possible, so that it recovers **general motifs that can be used to define and trace function, and to predict complexes**. The deliverable is therefore not a list of known PX sequences. It is a systematically built, curated sequence collection from which motifs (domain boundaries, linkers, binding interfaces, sequence signatures that separate subunit roles) can be derived, and which can then be used to recognise the same functional modules in sequences that annotation does not label.

Consequences for design decisions:
- Prefer recall over precision in the early stages; make every inclusion traceable (source, evidence, class), so that unnecessary inclusions can be removed later by curation, not by silent filtering.
- Domain annotation is a *starting hook*, not the definition of the family. The component-binding domain (CBD) is **not an annotated domain in any database**; it is found only through the PSBD (PF02817) hit next to it. Profiles built from the curated set must therefore extend beyond the annotated PSBD.
- The end use is functional: which module binds which partner (E1, E3, other), which proteins can form which complex, and how that varies across fungi. Sequence signatures that separate E1-binding from E3-binding PSBDs are a central target, because the PSBD alone does not say which partner it binds (in mammalian PDC the E2 PSBD binds E1 and the E3BP PSBD binds E3).
- **Suggestion (not agreed):** complex prediction could later use co-occurrence of subunit types per genome, motif-based interface signatures, and structure prediction of selected pairs. Nothing for this exists in the repo yet.

## 2. Domain vocabulary used in the code

| Letter | Meaning | Entries |
|---|---|---|
| L | lipoyl / biotin-lipoyl attachment | IPR000089, PF00364, IPR003016, PROSITE PS50968 |
| P | E3-binding / peripheral subunit-binding domain (PSBD) | PF02817 (this *is* the Pfam entry for the PSBD; there is no separate one), IPR004167, IPR036625 (superfamily), Gene3D G3DSA:4.10.320.10, SUPFAM SSF47005 |
| C | 2-oxoacid acyltransferase catalytic domain | IPR001078, PF00198 |
| family | whole-protein family | IPR045257 (E2 / PDX1, PANTHER PTHR23151), IPR006255 SucB, IPR006257 LAT1, IPR050537; PANTHER subfamily PTHR23151:SF82 "PDC protein X component" |
| broad | generic superfamily entries that match unrelated proteins | IPR011053, IPR023213 (excluded from fetching by default) |

Preliminary classes from architecture strings: L-P-C (E2-like), L-P (E3BP / protein X-like), L-C (2-oxoacid E2 without PSBD), P only ("PSBD only"), P-C, L only, C only. **The user checked these on real data:** nearly all "PSBD only" proteins are PX (legitimately without a lipoyl hit, but with a CBD that no database annotates); "lipoyl only" are few and show no clear PX signature; "P-C" may be interesting as a possible mammalian-core-substituting PX if its PSBD binds E3 rather than E1. These classes are labels from annotation, not functional assignments.

## 3. The strategy in one paragraph

Strategy 1 (see `docs/strategy_1.md`): (1a) an annotation net on UniProtKB through InterPro, centred on the E3-binding domain and the family entry, with lipoyl and catalytic entries used only to annotate, so architectures stay complete; (1b) the same net on UniParc, because many real members have no UniProtKB entry (RefSeq-only, WGS, MAG); (2) inspect and curate by architecture and cluster; (3) build profiles from the curated clusters and search databases that annotation cannot reach (`nr` fungi, optionally tblastn on fungal WGS contigs); (4) feed new hits back and repeat. The user's first target after 1b is to cut out all potential CBDs, align and cluster them, and build one or more fungal CBD HMMs (compared with the general PF02817 HMM).

## 4. Hard-won facts and traps

- **Names are not evidence.** The yeast gene symbol PDX1 is protein X; pyridoxal synthase is also PDX1. Genome pipelines therefore name many PX proteins "Pyridoxine biosynthesis protein". `name_note` in `proteins.tsv` flags them. Classify by domains and sequence.
- **Annotation can miss true members**, in two different ways:
  - *No UniProtKB entry*: UPI001445F4C4 (*Daldinia childiae*, RefSeq XP_033438179, full Pfam PF00364 + PF02817) is a clear PX that only UniParc has. UPI0044A48F9C (*Peltula* sp. TS41687, WGS KAI9786095, a "MAG:" name) has only Gene3D and SUPERFAMILY matches (IPR036625), no Pfam.
  - *No detectable signature*: diverged members; this is why stage 3 (sequence search from curated profiles) exists.
- **"MAG:" prefix** means metagenome-assembled genome; the taxon is a bin label and may be wrong. Keep a flag; confirm single-MAG candidates by neighbouring genes before counting them as a lineage.
- **InterPro API** caps pages at 200 proteins whatever `page_size` is; the server was slow (about 20 s per page of 200). Hence the fetch is done without sequences and the sequences come from UniProt in batches (`SEQUENCES_FROM_UNIPROT`). The InterPro `entries` field of a protein lists only the *queried* entry, so a protein's full architecture needs the annotate-only queries (or UniParc's match list).
- **UniParc search** (`rest.uniprot.org/uniparc/search`): `feature_id:<ID>` and `taxonomy_name:Fungi` work (taxonomy name is lineage-based and finds descendants); `organism_id:4751` matches only the exact taxon; `NOT uniprotkb:*` is rejected, so UniProtKB membership is decided client-side. There is no separate Pfam/InterPro field. Result fields exist for Pfam, Gene3D, SUPFAM, PANTHER and others. **The UniParc step in `Family_fetch_strategy1.ipynb` (step 4b) has only been tested against a mock**; JSON key names follow UniParc's single-entry JSON and `feature_id` behaviour on a full run is unconfirmed.
- Sandbox limits: the cloud shell cannot reach UniProt/InterPro directly (403 from the proxy); only the web-fetch tool can, in summarised form. Live testing therefore happens on the user's machine, not in the agent session. Treat every live-API behaviour as unconfirmed until the user has run it.

## 5. What is built and what is verified

| Piece | State |
|---|---|
| A_build_databases | cross-platform; tested on small mock data on Linux; not at UniRef scale |
| B_query_to_tree | still has Linux-only shell calls; not tested at scale |
| C_cluster_growth | competitive profile growth from anchors; tested on mock data (Linux); not tested on Windows or at scale |
| Family_fetch | InterPro fetch, union, architectures, per-architecture FASTA files, reference-set check (Venn, missing list, optional inclusion), database build; live count requests confirmed by the user; the user has run the strategy-1 variant on real data |
| Family_fetch_strategy1 | net/annotate-only design, UniParc step 4b, `cbd_candidates.fasta`; UniParc step and the UniProt sequence batches untested live |

Test harness (not in the repo; in the agent scratchpad): a mock InterPro / UniProt / UniParc server and a notebook build script (`build.py`) that assembles notebooks from source parts. If the harness is gone, edit the `.ipynb` cells directly; keep the cell structure (markdown cell describing what happens, expected cost and parameters, followed by the code cell).

## 6. Working conventions the user expects

- **Document inside the notebook.** Every step needs a markdown cell saying what happens, what to expect (time/size, marked "not measured" if so), and which parameters matter. The user asked for this explicitly.
- **Verbosity in long steps.** Per-page progress, elapsed time, estimated time left, a heartbeat when waiting. The user wants to be able to see that nothing is hung.
- **No overclaiming.** Say what was only tested on mock data. Do not describe live behaviour that has not been seen.
- **Agree before building** when the user says so ("Don't synthesise before agreement"); the user wants to understand the design first (for example what the architecture step does).
- **Environments:** the user runs Jupyter on a Linux cluster (outbound internet needed for fetch steps), and has asked for Windows/Anaconda compatibility (MMseqs2 on Windows is only a beta; WSL2 is recommended in the notebooks). Colab works for Family_fetch. The repo is public: https://github.com/bforsbe/homology_clusters (GPL-3.0). Commit messages end with the attribution lines the session specifies.
- Output tables are plain TSV (tab-separated text) so they open in Excel/pandas.

## 7. Priorities, in order

1. **Run `Family_fetch_strategy1.ipynb` live with UniParc on**, inspect `architectures/architectures.tsv`, the UniParc table (entries found, already in union, added) and `cbd_candidates.tsv`; fix whatever the real API does differently. Check that UPI001445F4C4 and UPI0044A48F9C appear.
2. **Curate the UniProtKB + UniParc set** (remove inclusions that do not belong) before any widening. Add class/architecture exclusion options if the user asks.
3. **CBD characterisation:** cut the CBD region (PSBD plus flank, size decided from the alignment), align (deep alignment advice: align domain regions separately, `hmmalign` to a seed HMM, MAFFT `--retree 2`/`--parttree`, FAMSA; clustering needs no alignment), cluster all potential CBDs, build one or more fungal CBD HMMs; compare with the general PF02817 HMM.
4. **Separate E2 / OGDC-BCKDC E2 / E3BP** with notebook C using curated anchors; target the sequence signatures that distinguish E1-binding from E3-binding PSBDs.
5. **Stage 3 sequence net:** profiles from curated clusters against `nr` restricted to fungi (includes WGS and MAG translations), optionally tblastn on fungal WGS contigs. First a cheap manual test: PSI-BLAST against `nr` (Fungi) from a PSSM of several divergent members (not a single consensus: PSI-BLAST drifts toward close matches). Count how many hits are not in the union to decide whether a local `nr` subset is worth the download. Possible new notebook D.
6. **Motifs and complexes** (the user's end goal): derive general motifs from the curated clusters (CBD, lipoyl-domain and linker features, any conserved interface residues), define rules to trace function in unannotated sequences, and consider complex prediction. Scope this together with the user before building.
7. Housekeeping: notebook B Windows port, `createindex` for large databases, multi-query input in B, a column showing which source (Pfam / PANTHER / signature) each class rests on, PANTHER subfamily IDs as a query type.

## 8. Open questions for the user

- How wide should the first UniParc pass be (net IDs only, or also the lipoyl and catalytic IDs as separate nets)?
- Flank size around the PSBD for the CBD cut, and whether `P-C` proteins belong to the CBD set (the `CBD_CLASSES` parameter currently excludes them).
- Which taxa beyond fungi are useful as outgroups for motif definition (metazoa, plants, bacteria are one tick each in the notebook).
- What counts as "predicting complexes" for the user: subunit assignment from sequence, stoichiometry, or structural models.
