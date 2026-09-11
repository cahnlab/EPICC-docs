.. _configuration:

=============
Configuration
=============

For new users, it is recommended to use the local HTML builder at
``tools/epicc-builder.html`` (open directly in a browser — self-contained, no
internet connection required) to build and validate your sample metadata file
and choose analysis options. The tool exports a validated sample sheet and an
options YAML ready to pass to ``epicc run``.


Metadata sample file
====================

Summary
-------

Prepare your sample metadata file (start from the documented template at
``config/example_samples.tsv``) with the required 9 columns below. Pass it to
the pipeline via ``epicc run --samples your_samples.tsv``.

+-------------+-------+--------+--------+--------------+------------+-------------+-----------+---------+
| *Sample_ID* | *Assay* | *Genome* | *Levels* | *Replicate_ID* | *Read_files* | *Read_layout* | *IP_target* | *Control* |
+=============+=======+========+========+==============+============+=============+===========+=========+
+-------------+-------+--------+--------+--------------+------------+-------------+-----------+---------+

- **Sample_ID**: Unique identifier for the sample, used in output filenames.
  Must be filesystem-safe — no ``__``, ``/``, whitespace, or shell metacharacters.

- **Assay**: Type of assay. Must be one of:
  ``ChIP_broad``, ``ChIP_narrow``, ``CUT_RUN_broad``, ``CUT_RUN_narrow``,
  ``CUT_TAG_broad``, ``CUT_TAG_narrow``, ``ATAC``, ``RNAseq``, ``RAMPAGE``,
  ``sRNA``, ``WGBS``, ``WGBS_nd``, ``PBAT``, ``EMseq``, ``dmC``

- **Genome**: Reference genome name (e.g. ``ColCEN``, ``Spombe``). Must match
  an entry under ``genomes:`` in the options file.

- **Levels**: Comma-separated ``factor:level`` pairs describing experimental
  conditions (e.g. ``genotype:WT,tissue:leaf``). All samples in the file must
  use the same factor names.

- **Replicate_ID**: Replicate identifier (e.g. ``rep1``, ``rep2``). All
  samples sharing the same Assay, Levels, IP_target, and Genome are merged
  for downstream analysis.

- **Read_files**: Path to input data. Accepts:

  - SRA accession (downloaded automatically): ``SRR27821931``
  - Multiple SRA accessions merged: ``SRR27821931+SRR27821932``
  - Local FASTQ path (SE): ``/archive/fastq/sample.fq.gz``
  - Local FASTQ paths (PE, comma-separated): ``/archive/fastq/sample_R1.fq.gz,/archive/fastq/sample_R2.fq.gz``
  - HTTP(S) URL to FASTQ, BAM, or bedMethyl file
  - Local BAM path (for ``dmC`` or ``aligned_bams`` mode)

- **Read_layout**: ``PE`` for paired-end data or ``SE`` for single-end data. For SRA
  accessions this must match the run's real layout: the download rule checks what
  ``fasterq-dump`` produced and stops with the accession name if the two disagree,
  rather than carrying an empty FASTQ through mapping.

- **IP_target**: Required for ChIP and CUT&x assays. The antibody target
  (e.g. ``H3K9me2``, ``TB1``) for IP samples, or the control type
  (e.g. ``Input``, ``WCE``, ``IgG``) for control samples. Leave blank for
  non-IP assays.

- **Control**: Sample_ID of the control sample for this IP (e.g. the Input or
  IgG sample's Sample_ID). Required for ChIP and CUT&x IP samples. Leave blank
  for control samples and non-ChIP assays.

A migration script is available to convert old-format sample sheets:

.. code-block:: bash

   python scripts/migrate_sample_sheet.py old_samples.tsv -o new_samples.tsv


Columns common to all sample types
-----------------------------------

- **Sample_ID** and **Genome**: see above.

- **Levels**: The combination of all factor levels is the basis for pairwise
  comparisons (e.g. ``genotype:WT,tissue:root`` vs ``genotype:ddm1,tissue:root``).
  All samples must have the same set of factor names.

- **Replicate_ID**: Any value to match replicates of the same condition
  (e.g. ``rep1``, ``repA``, ``1``). All replicates with matching Assay, Levels,
  IP_target, and Genome are merged for analysis-level outputs.

- **Read_files**: See above. HTTP(S) URLs are supported for FASTQ, BAM, and
  bedMethyl inputs.

- **Read_layout**: ``PE`` or ``SE``.

Migrating from the old sample-sheet format
-------------------------------------------

Sample sheets from older versions of EPICC used a different column schema
(``data_type``, ``line``, ``tissue``, ``sample_type``, ``replicate``,
``seq_id``, ``fastq_path``, ``paired``, ``ref_genome``). To convert an
existing legacy sample sheet to the current 9-column format, run:

.. code-block:: bash

   python scripts/migrate_sample_sheet.py old_samples.tsv -o new_samples.tsv

The converter maps each old column to its current equivalent (e.g.
``line``/``tissue`` are joined into the ``Levels`` column as
``genotype:<line>,tissue:<tissue>``) and writes the new sheet to the path
given by ``-o``.


Columns specific to each assay type
------------------------------------

ChIP-seq — Histones and Transcription Factors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- **Assay**: ``ChIP_broad`` for histone marks with broad peaks (e.g.
  H3K9me2, H3K27me3) or ``ChIP_narrow`` for marks with narrow peaks
  (e.g. H3K4me3) and transcription factors. The
  `ENCODE histone ChIP-seq target categorization
  <https://www.encodeproject.org/chip-seq/histone/>`__ is a good
  starting point: broad-domain marks (H3K27me3, H3K36me3, H3K9me1/2,
  H3K79me2/3, H4K20me1, H3K4me1) use ``ChIP_broad``; punctate marks
  and TFs (H3K4me2/3, H3K27ac, H3K9ac, H2AFZ) use ``ChIP_narrow``.
  Note that H3K27ac is narrow despite being on the same residue as
  the broad mark H3K27me3. H3K9me3 is nominally broad but heavily
  enriched in repeats — see ENCODE's note. Some targets profit from
  running both passes and comparing: RNA Pol II has sharp TSS peaks
  plus broader gene-body coverage, and H3K27me3 Polycomb domains
  can contain internal islands.

- **IP_target**: Required for all ChIP samples including controls. The name
  of what was pulled down (e.g. ``H3K9me2`` or ``TB1`` for IPs; ``Input``,
  ``WCE``, or ``IgG`` for controls).

- **Control**: For IP samples, the Sample_ID of the control sample. Multiple
  IP samples can share the same control.

To use different controls for different marks (e.g. H3 vs H4 pulldown as
controls for different IPs), assign the appropriate control Sample_ID in the
Control column for each IP row.

Example (histone):

.. code-block:: text

   WT_leaf_H3K9me2_rep1  ChIP_broad  ColCEN  genotype:WT,tissue:leaf  rep1  SRR12345  PE  H3K9me2  WT_leaf_Input_rep1
   WT_leaf_Input_rep1    ChIP_broad  ColCEN  genotype:WT,tissue:leaf  rep1  SRR12346  PE  Input

Example (transcription factor):

.. code-block:: text

   WT_leaf_TB1_rep1  ChIP_narrow  ColCEN  genotype:WT,tissue:leaf  rep1  SRR12347  PE  TB1  WT_leaf_Input_rep1

Differential nucleosome sensitivity (DNS-seq) can be analyzed with
``ChIP_broad``, using ``MNase`` as IP_target for the light digest and
``Input`` for the heavy digest.

CUT&RUN and CUT&Tag
^^^^^^^^^^^^^^^^^^^^

- **Assay**: ``CUT_RUN_broad`` / ``CUT_TAG_broad`` for diffuse marks
  (H3K27me3, H3K9me2, etc.); ``CUT_RUN_narrow`` / ``CUT_TAG_narrow`` for
  sharp marks and TFs (H3K4me3, CTCF, etc.). All four assay types route
  through the ChIP conda environment.

- **IP_target**: Required for all CUT&x samples including controls. Use the
  antibody target for IPs and ``IgG`` for controls.

- **Control**: For IP samples, the Sample_ID of the IgG control. Multiple IPs
  commonly share a single IgG (the typical CUT&RUN convention is one IgG per
  batch, not per replicate).

- **Peak callers**: Defaults are shape-aware — ``*_broad`` → epic2;
  ``*_narrow`` → SEACR. MACS2 is also supported. Override via
  ``cut_callpeaks.broad_caller`` / ``cut_callpeaks.narrow_caller`` in the
  options file.

Example (CUT&RUN sharing one IgG across replicates):

.. code-block:: text

   WT_endo_H3K27me3_rep1  CUT_RUN_broad  ColCEN  genotype:WT,tissue:endosperm  rep1  SRR8310960  PE  H3K27me3  WT_endo_IgG_rep1
   WT_endo_H3K27me3_rep2  CUT_RUN_broad  ColCEN  genotype:WT,tissue:endosperm  rep2  SRR8310958  PE  H3K27me3  WT_endo_IgG_rep1
   WT_endo_IgG_rep1       CUT_RUN_broad  ColCEN  genotype:WT,tissue:endosperm  rep1  SRR8310961  PE  IgG

ATAC-seq
^^^^^^^^^

- **Assay**: ``ATAC``
- **IP_target**: Leave blank.
- **Control**: Leave blank.

RNA-seq
^^^^^^^^

- **Assay**: ``RNAseq``
- **IP_target**: Leave blank.
- **Control**: Leave blank.

RAMPAGE
^^^^^^^^

- **Assay**: ``RAMPAGE``
- **IP_target**: Leave blank.
- **Control**: Sample_ID of the corresponding RNAseq sample (used for
  normalization).

small RNA-seq
^^^^^^^^^^^^^^

- **Assay**: ``sRNA``
- **IP_target**: Leave blank.
- **Control**: Leave blank.

Whole Genome Bisulfite Sequencing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- **Assay**: One of ``WGBS``, ``WGBS_nd``, ``PBAT``, or ``EMseq``. These
  labels identify the conversion chemistry and inform Bismark alignment
  parameters:

  - ``WGBS`` — standard directional WGBS (e.g. TruSeq, Swift HTP)
  - ``WGBS_nd`` — non-directional WGBS (e.g. Zymo Pico Methyl-Seq, Swift Accel-NGS)
  - ``PBAT`` — post-bisulfite adapter tagging
  - ``EMseq`` — enzymatic methyl-seq (NEB EM-seq kit)

- **IP_target**: Leave blank.
- **Control**: Leave blank.

Direct Methylation from Long-Read Sequencing (dmC)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- **Assay**: ``dmC``. For samples with native base modifications (Oxford
  Nanopore, PacBio) that have not undergone bisulfite conversion.

- **Read_files**: Path to input file. Supports:

  - **modBAM**: BAM files with MM/ML methylation tags from basecalling
    (e.g. Dorado, Guppy)
  - **bedMethyl**: Pre-computed methylation calls in bedMethyl format
    (e.g. from ``modkit pileup``)

  The pipeline automatically detects the format. modBAM files are aligned
  and processed through ``modkit pileup``. Both formats are converted to
  Bismark-compatible CX_report format for downstream analysis.

- **Read_layout**: Only ``SE`` is currently supported.
- **IP_target**: Leave blank.
- **Control**: Leave blank.


Configuration file
==================

Summary
-------

Update ``config/epicc-options.yaml`` with your paths and parameters. The file
is commented inline; all commonly adjusted options are described below. Pass a
different options file via ``epicc run --options``.

Repository folder
^^^^^^^^^^^^^^^^^

``repo_folder`` is optional. The pipeline auto-detects the repository location
from the Snakefile at runtime. Override explicitly only when accessing the
repository via a non-standard path.

Analysis name
^^^^^^^^^^^^^

``analysis_name`` labels combined output files (e.g. matrices, plots, reports).

Sample file
^^^^^^^^^^^

The path to the sample metadata file. Either set ``sample_file:`` in the
options file or pass ``epicc run --samples FILE`` on the command line (the CLI
flag takes precedence).

Reference genome files
^^^^^^^^^^^^^^^^^^^^^^

For each reference genome name used in the sample file, add an entry under
``genomes:`` in the options file. All ``*_file`` fields accept local paths
(absolute or relative to the repository root) or HTTP(S) URLs; gzipped inputs
are handled transparently.

.. code-block:: yaml

   genomes:
     ColCEN:
       genus: "Arabidopsis"
       species: "thaliana"
       fasta_file: "path/to/ColCEN.fa.gz"          # .fa/.fasta(.gz)
       gff_file: "path/to/ColCEN_genes.gff3.gz"    # .gff*(.gz)
       #gtf_file: "path/to/ColCEN.gtf"             # optional; auto-derived from GFF via gffread if omitted
       te_file: "path/to/ColCEN_TE.gff3.gz"        # optional; .bed(.gz) or .gff3(.gz)
       gaf_file: "data/ColCEN_infoGO.tab.gz"        # only required when GO: true
       gene_info_file: "data/ColCEN_genes_info.tab.gz"  # only required when GO: true
       ncbi_taxid: "3702"
       #genomesize: 1.3e8         # optional; auto-computed from FASTA if omitted
       #star_index: "--genomeSAindexNbases 12"  # optional; auto-computed if omitted
       #structural_rna_fafile: "<auto>"         # optional; auto-derived via Infernal/Rfam if omitted

Several fields are auto-derived at runtime and do not need to be specified:
``gtf_file`` (generated from GFF via gffread), ``genomesize`` (total non-N
bases in FASTA), ``star_index`` parameters, ``structural_rna_fafile``
(fetched from Rfam via Infernal), and ``ncbi_taxid`` (looked up via the NCBI
Datasets CLI). Any value provided explicitly in the options file overrides the
computed value.

The ``te_file`` field accepts ``.bed(.gz)`` (used as-is) or ``.gff3(.gz)``
(auto-converted to BED6 using the GFF3 ``ID=`` attribute as the name column).
The name of each feature must be unique.

The GO database for each genome is auto-derived as
``org.<G><species>.eg.db`` (e.g. ``org.Athaliana.eg.db``) and installed into
and loaded from ``genomes/<refgenome>/GO/`` to prevent collisions when
multiple genomes of the same species are configured (e.g. ColCEN and TAIR10).

The old bare-key format (genome blocks as top-level keys with a separate
species block) is still accepted but triggers a deprecation warning at startup.
Migrate to the ``genomes:`` namespace shown above.

Analysis parameters
^^^^^^^^^^^^^^^^^^^

Full documentation is available in ``config/epicc-options.yaml`` (all options
are commented inline) and in the epicc-builder tools.

Resources and Profiles
======================

Summary
-------

Resources are pre-defined based on conservative values for optimal
time/resource requirements. Review and adjust them to your system before
running:

- Rule-specific resource requirements: in ``config/epicc-options.yaml``
- Profile configuration: in the profile directory for your scheduler
  (e.g. ``profiles/slurm/config.yaml`` for SLURM)

Rule-specific resource requirements
------------------------------------

Each rule has been assigned resource sets based on its computational demands.
Edit the resource sets in the options file or override individual rules using
``set-resources:`` in the profile. Each set defines threads, memory (in MB),
and optionally a ``runtime`` (in minutes for SLURM):

.. code-block:: yaml

   low_resources:
     threads: 1
     mem_mb: 1000

   standard_resources:
     threads: 4
     mem_mb: 2000

   heavy_resources:
     threads: 8
     mem_mb: 16000

   max_resources:
     threads: 16
     mem_mb: 32000

   single_thread:
     threads: 1
     mem_mb: 32000

Profiles
--------

Four profiles are included:

- ``profiles/default/`` — workflow-level per-rule resource and thread defaults (used automatically)
- ``profiles/slurm/`` — SLURM cluster (sbatch)
- ``profiles/uge/`` — UGE cluster (qsub)
- ``profiles/geno/`` — example profile for additional scheduler types

Pass the appropriate profile via ``epicc run --profile profiles/slurm``.

If you need a cluster Quality-of-Service (QoS) setting for specific rules,
add ``slurm_extra: "'--qos=<your_qos>'"`` under the relevant rule in
``set-resources:`` within the profile. The ``snakemake-executor-plugin-slurm``
maps the ``qos`` resource key directly to ``--qos=``.

For schedulers not covered by the included profiles, copy the closest existing
profile, adapt it for your site, and install the corresponding Snakemake
executor plugin (see `executor plugin catalog <https://snakemake.github.io/snakemake-plugin-catalog/index.html>`__).


Output options
==============

Default parameters
^^^^^^^^^^^^^^^^^^

- **Full analysis**:
  When ``false``, only mapping and bigwig generation occur. When ``true``
  (default), single-data analyses (peak calling, differential expression, DMRs)
  and combined analyses (Upset plots, heatmaps, metaplots) are also performed.
  Change ``full_analysis`` in the options file.

- **Limited QC output**:
  ``QC_option`` controls FastQC reporting. ``"none"`` (default) skips FastQC
  entirely; ``"all"`` runs FastQC on all raw and trimmed FASTQ files.

- **No Gene Ontology analysis**:
  GO analysis is OFF by default (``GO: false``) because it requires manually
  building a database. Set ``GO: true`` and provide ``gaf_file`` and
  ``gene_info_file`` for the genome. See
  :ref:`Help GO <ref-help-go-analysis>` for database construction details. Pre-built
  databases are available for Arabidopsis thaliana (TAIR10 / ColCEN) and Maize
  B73 (v5 / NAM) in the ``data/`` folder.

- **No TE analysis**:
  By default, no transposable element analysis is performed
  (``te_analysis: false``). When ``true``, additional combined-analysis
  heatmaps and metaplots are generated over the configured ``te_file``
  annotation in addition to the standard gene-centered plots. Requires
  ``te_file`` to be set for the corresponding genome.

- **For ChIP-seq and ATAC-seq**:
  The default aligner is Chromap (~10x faster than Bowtie2). Change
  ``chip_aligner`` / ``atac_aligner`` to ``"bowtie2"`` for the traditional
  aligner. Mapping strategy options are available via
  ``chip_mapping_strategy`` (see `ChIP Mapping Parameters`_ below).

- **For sRNA-seq**:
  The default is not NextFlex v3 library preparation. If your data was made
  with this kit, an additional deduplication and trimming step is required.
  Set ``netflex_v3_deduplication: true`` in the options file. See `Known
  potential issues` in the README if you have mixed libraries.

  The default is not to filter structural RNAs prior to ShortStack analysis.
  Change ``structural_rna_depletion`` in the options file. While this step is
  recommended for small interfering RNA analysis, it requires a pre-built
  database of FASTA files. See :ref:`Help RFAM <ref-help-structural-rna>` to
  generate the required file before enabling this option. A pre-built file for
  Maize is available in the ``data/`` folder.

  The default is to only perform *de novo* microRNA identification
  (``--dn_mirna`` argument in ShortStack). To also annotate known microRNAs,
  download the FASTA file from `miRbase <https://www.mirbase.org>`__, filter
  it for your species, and add ``--known_miRNAs <path/to/known_miRNA_file.fa>``
  to the ``srna_mapping_params`` entry in the options file.


Configuration Options
^^^^^^^^^^^^^^^^^^^^^

Main output options
~~~~~~~~~~~~~~~~~~~

- ``full_analysis``: When ``false``, only mapping and bigwigs are produced.
  When ``true`` (default), single-data and combined analyses are also run.

- ``te_analysis``: When ``true``, generates additional heatmaps and metaplots
  over the ``te_file`` annotation. The feature name (4th column of BED, or
  ``ID=`` attribute for GFF3) must be unique. Default is ``false``.

- ``QC_option``: Controls FastQC reporting. ``"none"`` (default) skips FastQC;
  ``"all"`` runs FastQC on raw and trimmed FASTQs.

- ``plot_allreps``: When ``true``, all individual replicates are shown on
  heatmaps, metaplots, and browser shots. When ``false`` (default), each
  sample is represented by its merged replicates track.

Intermediate input formats
~~~~~~~~~~~~~~~~~~~~~~~~~~

- ``trimmed_fastqs``: When ``false`` (default), trimming is performed from
  raw FASTQs. Set to ``true`` if pre-trimmed FASTQs are provided.

- ``aligned_bams``: When ``true``, the pipeline expects pre-aligned BAM/SAM
  files in the ``Read_files`` column of the sample sheet rather than FASTQs.
  Currently supported for ChIP-seq assays only. No mapping-stats plot is
  available in this mode. Default is ``false``.

  These settings apply to *all* samples in the analysis. If you have a mix of
  raw data and intermediate files, run the pipeline in stages:

  1. Run once for the samples starting from raw data (optionally with
     ``full_analysis: false`` for less output).
  2. Add the samples with existing intermediate files to the sample sheet and
     update the corresponding options.
  3. Run normally again.

ChIP Mapping Parameters
~~~~~~~~~~~~~~~~~~~~~~~

Set via ``chip_mapping_strategy`` in the options file:

- ``default``: Standard mapping parameters
- ``repeat``: Centromere-specific mapping (more sensitive; forces Bowtie2 regardless of ``chip_aligner``)
- ``repeatall``: Centromere mapping with relaxed MAPQ (forces Bowtie2)
- ``all``: Relaxed mapping parameters

DMRs parameters
~~~~~~~~~~~~~~~

DNA methylation data is analyzed per context. The active contexts are
controlled by ``methylation_contexts`` (default: ``["CG", "CHG", "CHH"]``).
This option gates per-context bigwigs, DMR calls, and PCA plots.

For animal genomes where non-CpG methylation is negligible, set:

.. code-block:: yaml

   methylation_contexts: ["CG"]

to skip empty CHG/CHH outputs. Subcontexts (CAG, CAA, etc.) are not yet
supported.

DMRs are called with the R package
`DMRcaller <https://www.bioconductor.org/packages/release/bioc/html/DMRcaller.html>`__
(DOI: 10.18129/B9.bioc.DMRcaller) for each configured context, using
``computeDMRsReplicates`` with beta-regression and the following (stringent)
default parameters:

.. code-block:: text

   CG:  method="noise-filter", binSize=200, test="score", pValueThreshold=0.01,
        minCytosinesCount=5, minProportionDifference=0.3, minGap=200, minSize=50, minReadsPerCytosine=3
   CHG: method="noise-filter", binSize=200, test="score", pValueThreshold=0.01,
        minCytosinesCount=5, minProportionDifference=0.2, minGap=200, minSize=50, minReadsPerCytosine=3
   CHH: method="bins",         binSize=200, test="score", pValueThreshold=0.01,
        minCytosinesCount=5, minProportionDifference=0.1, minGap=200, minSize=50, minReadsPerCytosine=3

These parameters were selected based on the results of
`Catoni et al. 2018 <https://academic.oup.com/nar/article/46/19/e114/5050634>`__.

A custom parameter sweep mode is available. Toggle ``custom_script_dmrs: true``
in the options file to use it.

In-line customization
~~~~~~~~~~~~~~~~~~~~~

Any configuration option can be overridden on the command line by passing it
after ``--`` to ``epicc run``, using Snakemake's ``--config`` flag:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv -- --config chip_mapping_strategy="repeatall"
