.. _configuration:

=============
Configuration
=============

For new users, it is recommended to use the local HTML builder at
``tools/epicc-builder.html`` (open directly in a browser; it runs locally, and
loads its Tabulator and js-yaml libraries from a CDN, so the page needs internet
access) to build and validate your sample metadata file and choose analysis
options. The tool exports a validated sample sheet and an
options YAML ready to pass to ``epicc run``.


Metadata sample file
====================

Summary
-------

Prepare your sample metadata file (start from the documented template at
``config/example_samples.tsv``). It is a tab-separated file (TSV) with a header
row and 11 columns; each row defines one sample. Pass it to the pipeline via
``epicc run --samples your_samples.tsv``.

.. code-block:: text

   Sample_ID  Assay  Genome  Levels  Replicate_ID  Read_files  Read_layout  IP_target  Control  Peak_type  Comments

The four trailing columns (``IP_target``, ``Control``, ``Peak_type``,
``Comments``) may be left blank or omitted from the right-hand end of a row; a
WGBS row can stop after ``Read_layout``. Lines starting with ``#`` are skipped,
so rows can be parked without deleting them. Trailing whitespace on any field is
stripped.

- **Sample_ID**: Unique identifier for the sample, used in output filenames and
  as the target of the ``Control`` column. Must be filesystem-safe — no ``__``,
  ``/``, whitespace, or shell metacharacters.

- **Assay**: The experimental method only. Must be one of:
  ``ChIP``, ``CUT_RUN``, ``CUT_TAG``, ``ATAC``, ``RNAseq``, ``RAMPAGE``,
  ``sRNA``, ``WGBS``, ``WGBS_nd``, ``PBAT``, ``EMseq``, ``dmC``.
  Whether peaks are called as broad or narrow is set separately in
  ``Peak_type``; combined values such as ``ChIP_broad`` are not accepted in the
  sample sheet.

- **Genome**: Reference genome name (e.g. ``ColCEN``, ``Spombe``). Must match
  an entry under ``genomes:`` in the options file and must not contain ``__``.
  To map the same reads to several references, comma-separate them (e.g.
  ``B73,W22``); see :ref:`Mapping one sample to several genomes
  <ref-multi-genome>`.

- **Levels**: Comma-separated ``factor:level`` pairs describing experimental
  conditions (e.g. ``genotype:WT,tissue:leaf``). All samples in the file must
  use the same factor names, in the same order. Factor names and levels must
  not contain ``=``. The level values (not the factor names) are joined with
  ``_`` to form the label used in analysis-level file names (e.g. ``WT_leaf``).
  Give control samples (Input, WCE, IgG) meaningful Levels too, since Levels
  determine how samples are grouped.

- **Replicate_ID**: Replicate identifier (e.g. ``rep1``, ``rep2``). All
  samples sharing the same Assay, Levels, IP_target, and Genome are merged
  for downstream analysis. Control rows are merged on Levels, IP_target, and
  Genome only, so one Input/IgG/WCE can serve several IP types (for example
  broad and narrow, or ChIP and CUT&RUN) whatever Assay label the control rows
  carry.

- **Read_files**: Path to input data. Accepts:

  - SRA accession (downloaded automatically): ``SRR27821931``
  - Multiple inputs merged with ``+``: ``SRR27821931+SRR27821932``, or
    ``a.fq.gz+b.fq.gz``. ``+``-merge works for SRA accessions and FASTQ files
    (local or URL) and all components must be of the same type. For PE FASTQs,
    keep each pair together: ``a_R1.fq.gz,a_R2.fq.gz+b_R1.fq.gz,b_R2.fq.gz``.
    BAM and bedMethyl inputs cannot be merged this way; merge them upstream.
  - Local FASTQ path (SE): ``/archive/fastq/sample.fq.gz``
  - Local FASTQ paths (PE, comma-separated R1,R2): ``/archive/fastq/sample_R1.fq.gz,/archive/fastq/sample_R2.fq.gz``
  - HTTP(S) URL to a FASTQ, BAM, or bedMethyl file
  - Public S3 URI: ``s3://bucket/key/sample.bam``. Only public
    (authentication-free) objects are supported: there is no request signing
    and no credentials. Requester-Pays buckets (including SRA's
    Original-Format ``s3://sra-pub-src-*`` buckets) do not work; use the plain
    SRA accession instead.
  - Local BAM path (for ``dmC`` or ``aligned_bams`` mode)

  Local paths must exist before the run starts, and the same file or accession
  must not appear in more than one sample.

- **Read_layout**: ``PE`` for paired-end data or ``SE`` for single-end data. For SRA
  accessions this must match the run's real layout: the download rule checks what
  ``fasterq-dump`` produced and stops with the accession name if the two disagree,
  rather than carrying an empty FASTQ through mapping.

- **IP_target**: Required for ``ChIP``, ``CUT_RUN`` and ``CUT_TAG``, including
  control rows. The antibody target (e.g. ``H3K9me2``, ``TB1``) for IP samples,
  or the control type (e.g. ``Input``, ``WCE``, ``IgG``) for control samples.
  Leave blank for other assays.

- **Control**: Sample_ID of the control sample for this IP (e.g. the Input or
  IgG sample's Sample_ID). Required on ``ChIP``/``CUT_RUN``/``CUT_TAG`` IP rows
  for peak calling, and on ``RAMPAGE`` rows. Leave blank for plain control
  samples and other assays. Several IP samples can share one control. A
  control may itself declare a ``Control`` (a *dual-role* sample, see
  :ref:`ChIP-seq <ref-chip-dual-role>` below), but a chain can be at most two
  levels deep. A pulldown row with no ``Control`` is not peak-callable and is
  dropped from the peak and analysis targets.

- **Peak_type**: ``broad`` or ``narrow``, the peak-calling type. Required on
  ``ChIP``, ``CUT_RUN`` and ``CUT_TAG`` rows that call peaks, that is, rows with
  a ``Control``. Optional on a control-only row, which calls no peaks. Must be
  blank for every other assay (ATAC's peak type is fixed). It is an analytical
  choice kept separate from the assay, so the same libraries can be
  re-analyzed under the other peak type by editing this one column.

- **Comments**: Optional free text for your own notes (e.g. ``resequenced
  2026-03; higher duplication than rep2``). Never read by the pipeline and
  never part of analysis names or output paths. The column may be omitted
  entirely.

.. _ref-multi-genome:

Mapping one sample to several genomes
-------------------------------------

A comma-separated ``Genome`` (e.g. ``B73,W22``) maps one sample to every listed
reference. Reads are downloaded and trimmed once, using the bare ``Sample_ID``
(e.g. ``trim__WT_H3K9me2_rep1__R1.fastq.gz``), and then aligned to each
reference. From alignment onward every file name carries the genome as
``<Sample_ID>__<Genome>`` (e.g. ``final__WT_H3K9me2_rep1__B73.bam``), so the
analyses for different references never collide. Each genome needs its own
entry under ``genomes:`` in the options file and may not appear twice in the
list.

Analysis-level (merged-replicate) files use a derived name,
``{Assay}__{levels_label}__{IP_target}__{Genome}``, where ``Assay`` is combined
with the peak type for pulldown assays (e.g.
``ChIP_broad__WT__H3K9me2__Spombe``). This internal combined token appears in
output paths; it is not a valid sample-sheet value.


Columns specific to each assay type
------------------------------------

.. _ref-chip-dual-role:

ChIP-seq — Histones and Transcription Factors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- **Assay**: ``ChIP``.

- **Peak_type**: ``broad`` for histone marks with broad peaks (e.g.
  H3K9me2, H3K27me3) or ``narrow`` for marks with narrow peaks
  (e.g. H3K4me3) and transcription factors. The
  `ENCODE histone ChIP-seq target categorization
  <https://www.encodeproject.org/chip-seq/histone/>`__ is a good
  starting point: broad-domain marks (H3K27me3, H3K36me3, H3K9me1/2,
  H3K79me2/3, H4K20me1, H3K4me1) use ``broad``; punctate marks
  and TFs (H3K4me2/3, H3K27ac, H3K9ac, H2AFZ) use ``narrow``.
  Note that H3K27ac is narrow despite being on the same residue as
  the broad mark H3K27me3. H3K9me3 is nominally broad but heavily
  enriched in repeats — see ENCODE's note. Some targets profit from
  running both passes and comparing: RNA Pol II has sharp TSS peaks
  plus broader gene-body coverage, and H3K27me3 Polycomb domains
  can contain internal islands. To run both passes, copy the rows and
  change ``Peak_type``; the assay stays ``ChIP``.

- **IP_target**: Required for all ChIP samples including controls. The name
  of what was pulled down (e.g. ``H3K9me2`` or ``TB1`` for IPs; ``Input``,
  ``WCE``, or ``IgG`` for controls).

- **Control**: For IP samples, the Sample_ID of the control sample. Multiple
  IP samples can share the same control. A control may itself declare a
  ``Control``: in this *dual-role* case a sample is both another row's
  control and an analysis target in its own right (e.g. H3 serving as
  H3K9me2's control while itself being normalized against Input). It gets the
  full analysis treatment (peaks, fold-change tracks, merged analysis group).
  Chaining stops there: the control's control must not declare one.

To use different controls for different marks (e.g. H3 vs H4 pulldown as
controls for different IPs), assign the appropriate control Sample_ID in the
Control column for each IP row.

Example (histone):

.. code-block:: text

   WT_leaf_H3K9me2_rep1  ChIP  ColCEN  genotype:WT,tissue:leaf  rep1  SRR12345  PE  H3K9me2  WT_leaf_Input_rep1  broad
   WT_leaf_Input_rep1    ChIP  ColCEN  genotype:WT,tissue:leaf  rep1  SRR12346  PE  Input

Example (transcription factor):

.. code-block:: text

   WT_leaf_TB1_rep1    ChIP  ColCEN  genotype:WT,tissue:leaf  rep1  SRR12347  PE  TB1    WT_leaf_Input_rep1  narrow
   WT_leaf_Input_rep1  ChIP  ColCEN  genotype:WT,tissue:leaf  rep1  SRR12348  PE  Input

Example (dual-role control; H3 is normalized against Input and also serves as
the control for H3K9me2):

.. code-block:: text

   Input_rep1     ChIP  ColCEN  genotype:WT,tissue:leaf  rep1  SRR1  PE  Input
   H3_rep1        ChIP  ColCEN  genotype:WT,tissue:leaf  rep1  SRR2  PE  H3       Input_rep1  broad
   H3K9me2_rep1   ChIP  ColCEN  genotype:WT,tissue:leaf  rep1  SRR3  PE  H3K9me2  H3_rep1     broad

Differential nucleosome sensitivity (DNS-seq) can be analyzed with
``Assay=ChIP`` and ``Peak_type=broad``, using ``MNase`` as IP_target for the
light digest and ``Input`` for the heavy digest.

CUT&RUN and CUT&Tag
^^^^^^^^^^^^^^^^^^^^

- **Assay**: ``CUT_RUN`` or ``CUT_TAG``. Both route through the ChIP conda
  environment.

- **Peak_type**: ``broad`` for diffuse marks (H3K27me3, H3K9me2, etc.);
  ``narrow`` for sharp marks and TFs (H3K4me3, CTCF, etc.).

- **IP_target**: Required for all CUT&x samples including controls. Use the
  antibody target for IPs and ``IgG`` for controls.

- **Control**: For IP samples, the Sample_ID of the IgG control. Multiple IPs
  commonly share a single IgG (the typical CUT&RUN convention is one IgG per
  batch, not per replicate). One biological IgG can serve broad and narrow IPs,
  and CUT&RUN and CUT&Tag IPs, at once.

- **Peak callers**: Defaults are shape-aware — ``broad`` → epic2;
  ``narrow`` → SEACR. MACS2 is also supported. Override via
  ``cut_callpeaks.broad_caller`` / ``cut_callpeaks.narrow_caller`` in the
  options file, or ``--cut-broad-caller`` / ``--cut-narrow-caller`` on the
  command line.

Example (CUT&RUN sharing one IgG across replicates):

.. code-block:: text

   WT_endo_H3K27me3_rep1  CUT_RUN  ColCEN  genotype:WT,tissue:endosperm  rep1  SRR8310960  PE  H3K27me3  WT_endo_IgG_rep1  broad
   WT_endo_H3K27me3_rep2  CUT_RUN  ColCEN  genotype:WT,tissue:endosperm  rep2  SRR8310958  PE  H3K27me3  WT_endo_IgG_rep1  broad
   WT_endo_IgG_rep1       CUT_RUN  ColCEN  genotype:WT,tissue:endosperm  rep1  SRR8310961  PE  IgG

Example (CUT&Tag for a TF, single-end; SEACR is the default narrow caller):

.. code-block:: text

   WT_leaf_CTCF_rep1  CUT_TAG  hg38  genotype:WT,tissue:leaf  rep1  SRR12345  SE  CTCF  WT_leaf_IgG_rep1  narrow
   WT_leaf_IgG_rep1   CUT_TAG  hg38  genotype:WT,tissue:leaf  rep1  SRR12346  SE  IgG

ATAC-seq
^^^^^^^^^

- **Assay**: ``ATAC``
- **Peak_type**: Leave blank (peaks are always called as narrow).
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
  Bismark-compatible CX_report format for downstream analysis. BAM and
  bedMethyl inputs cannot be ``+``-merged.

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

For each reference genome name used in the ``Genome`` column of the sample
file, add an entry under the ``genomes:`` namespace in the options file. The
namespace is required; a missing ``genomes:`` block is a startup error. All
``*_file`` fields (``fasta_file``, ``gff_file``, ``te_file``, ``gaf_file``,
``gene_info_file``) accept local paths (absolute or relative to the repository
root) or HTTP(S) URLs, which are downloaded automatically when the rule runs;
gzipped inputs are handled transparently. A genome name must not contain
``__``.

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

Analysis parameters
^^^^^^^^^^^^^^^^^^^

Full documentation is available in ``config/epicc-options.yaml`` (all options
are commented inline) and in the epicc-builder tools. The most commonly tuned
settings can also be overridden per run with named ``epicc run`` flags (see
:ref:`Named overrides <ref-named-overrides>`); a flag takes precedence over the
options file.


.. _ref-resources-profiles:

Resources and Profiles
======================

Summary
-------

Resources are pre-defined based on conservative values for optimal
time/resource requirements. Review and adjust them to your system before
running:

- Rule-specific resource requirements: in ``profiles/default/config.yaml``
  (the workflow profile, passed automatically; override with
  ``epicc run --workflow-profile DIR``)
- Cluster execution settings: in the profile for your scheduler
  (e.g. ``profiles/slurm/config.yaml`` for SLURM)

Rule-specific resource requirements
------------------------------------

Each rule is bound to a resource tier in the ``set-resources:`` block of
``profiles/default/config.yaml``, with thread counts under ``set-threads:``.
A tier defines memory (in MB) and a runtime, for example:

.. code-block:: yaml

   low_resources: &low
     mem_mb: 1000
     runtime: "1h"

   heavy_resources: &heavy
     mem_mb: 16000
     runtime: "8h"

   max_resources: &max
     mem_mb: 32000
     runtime: "2d"

Cluster profiles override these values: ``set-resources:`` and
``set-threads:`` in a cluster profile take precedence over the workflow
profile, so site-specific tuning is additive.

Profiles
--------

The following profiles are included under ``profiles/``:

- ``default/`` — workflow-level per-rule resource and thread defaults (used automatically)
- ``slurm/`` — SLURM cluster (sbatch) template
- ``uge/`` — UGE/SGE cluster (qsub) template

Pass the cluster profile via ``epicc run --profile profiles/slurm``. If
``--profile`` and ``--cores`` are both omitted, ``epicc`` uses
``profiles/slurm`` when ``sbatch`` is available and otherwise runs locally on
half of the available cores.

To keep a site profile outside the repository (the usual choice for
conda-installed copies, and for sharing one profile across projects), install
a template with ``epicc init-profile``:

.. code-block:: bash

   epicc init-profile slurm my-cluster    # SLURM
   epicc init-profile uge my-cluster      # UGE/SGE

This copies the template to ``~/.config/snakemake/my-cluster/config.yaml``.
Edit every line marked ``EDIT:`` (SLURM account, per-rule resource overrides)
and run with ``--profile my-cluster``. Use ``--force`` to overwrite an existing
profile. ``epicc`` stops before submitting if an active line of the profile
still contains an unedited ``<your-...>`` placeholder.

The shipped SLURM profile sets:

- ``jobs: 16`` — number of concurrent cluster jobs
- ``retries: 1`` — absorbs transient SLURM control-plane glitches and rare
  port collisions in the pooled-fallback DMR path. Keep ``retries`` at 1 or
  higher in site profiles derived from it.
- ``slurm-requeue: true`` — leaves node failures and preemption to SLURM's
  requeue, so the plugin keeps tracking the job under its original ID. Without
  it, on a cluster with ``JobRequeue=1``, a ``NODE_FAIL`` is reported as a
  failure and ``retries`` submits a second copy while SLURM's requeued one
  runs; the two overwrite each other's outputs. Do not use
  ``slurm-no-requeue``: a preempted job is then cancelled and the plugin waits
  on it indefinitely.
- ``default-resources: runtime="1h"`` — fallback wall time (60 minutes) for a
  rule that does not declare its own. Rules override it with their own
  ``runtime``; raise it, or add per-rule overrides under ``set-resources``, if
  your cluster has tighter limits or longer jobs.

Memory units are in MB. If you need a Quality-of-Service (QoS) setting or other
site-specific sbatch flags for specific rules, add
``slurm_extra: "'--qos=<your_qos>'"`` under the relevant rule in
``set-resources:`` within the profile.

Run ``epicc run`` from a login or development node, not from inside an
``sbatch`` job. Under an allocation, ``snakemake-executor-plugin-slurm``
deletes every ``SLURM_*`` environment variable, including ``SLURM_CONF``, and
on clusters where that variable is the only route to ``slurm.conf`` every
submitted job then fails with ``srun: fatal: Could not establish a
configuration source``. If submitting from within a job is unavoidable, restore
the path with ``precommand: 'export SLURM_CONF=/path/to/slurm.conf'`` in the
profile. A second, independent cause is
``snakemake-executor-plugin-slurm-jobstep`` 0.6.1, which strips ``SLURM_CONF``
inside the job wherever the run was launched; ``config/epicc-env.txt`` pins the
plugin below that release. To fix an existing environment:

.. code-block:: bash

   conda install -n epicc 'snakemake-executor-plugin-slurm-jobstep<0.6.1'

If conda environment creation fails inside allocations on your cluster, build
the environments first from the login node (see :ref:`Pre-building
environments <ref-build-envs>`) and then launch from the login node as well.

For schedulers not covered by the included profiles, copy the closest existing
profile, adapt it for your site, and install the corresponding Snakemake
executor plugin (see `executor plugin catalog <https://snakemake.github.io/snakemake-plugin-catalog/index.html>`__).

Temporary files are routed to the project filesystem by default; see
:ref:`TMPDIR routing <ref-tmpdir-routing>`.


Output options
==============

Default parameters
------------------

- **Full analysis**:
  When ``false``, only mapping and bigwig generation occur. When ``true``
  (default), single-data analyses (peak calling, differential expression, DMRs)
  and combined analyses (Upset plots, heatmaps, metaplots) are also performed.
  Change ``full_analysis`` in the options file, or pass
  ``--full-analysis`` / ``--no-full-analysis``.

- **Limited QC output**:
  ``QC_option`` controls FastQC reporting. ``"none"`` (default) skips FastQC
  entirely; ``"all"`` runs FastQC on all raw and trimmed FASTQ files.

- **No Gene Ontology analysis**:
  GO analysis is OFF by default (``GO: false``) because it requires manually
  building a database. Set ``GO: true`` (or pass ``--go``) and provide ``gaf_file`` and
  ``gene_info_file`` for the genome. See
  :ref:`Help GO <ref-help-go-analysis>` for database construction details. The
  two input files are provided in the ``data/`` folder for Arabidopsis thaliana
  (TAIR10 / ColCEN) and Maize B73 (v5). The database is named
  ``org.<G><species>.eg.db`` (e.g. ``org.Athaliana.eg.db``) and is installed
  into and loaded from ``genomes/<refgenome>/GO/``, so several references of
  one species (e.g. ColCEN and TAIR10) do not collide.

- **No TE analysis**:
  By default, no transposable element analysis is performed
  (``te_analysis: false``). When ``true``, additional combined-analysis
  heatmaps and metaplots are generated over the configured ``te_file``
  annotation in addition to the standard gene-centered plots. Requires
  ``te_file`` to be set for the corresponding genome. Also
  ``--te-analysis`` / ``--no-te-analysis``.

- **For ChIP-seq and ATAC-seq**:
  The default aligner is Bowtie2. Set ``chip_aligner`` / ``atac_aligner`` to
  ``"chromap"`` (or pass ``--chip-aligner`` / ``--atac-aligner``) for ~10x
  faster mapping. Chromap is recommended for SE-only datasets: on PE data only
  11-28% of reads are properly paired with Chromap versus 99%+ with Bowtie2,
  because Chromap maps the two ends independently and one mate is
  MAPQ-filtered when repeat paralogs are present. With the ``repeat`` or
  ``repeatall`` mapping strategy, Bowtie2 is used automatically regardless of
  the aligner setting. Mapping strategy options are available via
  ``chip_mapping_strategy`` (see `ChIP Mapping Parameters`_ below).

- **For RNA-seq**:
  Duplicate reads are not removed from mRNA-seq (``rna_deduplicate: false``):
  without UMIs, PCR and optical duplicates cannot be told apart from genuine
  high-expression reads, so deduplicating biases quantification. UMI-aware
  deduplication is not implemented. RAMPAGE is always deduplicated.
  Toggle with ``--rna-deduplicate`` / ``--no-rna-deduplicate``.

- **For sRNA-seq**:
  The default is not NextFlex v3 library preparation. If your data was made
  with this kit, an additional deduplication and trimming step is required.
  Set ``nextflex_v3_deduplication: true`` in the options file. This setting
  applies to all samples, so if you have a mix of library preparations, run the
  pipeline with each kind separately and then rerun with all the samples you
  want to analyze together.

  The default (``structural_rna_depletion: true``) is to filter structural RNAs
  (rRNAs, tRNAs, snoRNAs) before ShortStack analysis, which is recommended for
  microRNA and small interfering RNA analysis. When ``structural_rna_fafile``
  is not set for the genome, the FASTA file is derived automatically with
  Infernal and Rfam (``infernal_threshold``: ``"ga"``, the Rfam gathering
  threshold, by default, or a numeric E-value such as ``1e-5`` for more
  aggressive filtering). To supply your own file, see
  :ref:`Help RFAM <ref-help-structural-rna>`. A pre-built file for Maize is
  available in the ``data/`` folder. Set ``structural_rna_depletion: false``
  to skip the step.

  The default is to only perform *de novo* microRNA identification
  (``--dn_mirna`` argument in ShortStack). To also annotate known microRNAs,
  download the FASTA file from `miRbase <https://www.mirbase.org>`__, filter
  it for your species, and add ``--known_miRNAs <path/to/known_miRNA_file.fa>``
  to the ``srna_mapping_params`` entry in the options file.


Configuration Options
---------------------

Main output options
^^^^^^^^^^^^^^^^^^^

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

- ``keep_intermediates``: Intermediate file retention after downstream rules
  finish (also ``epicc run --keep-intermediates TIER``):

  - ``standard`` (default): keeps per-replicate final BAMs; removes trimmed
    FASTQs, merged BAMs, and shifted BAMs.
  - ``none``: removes all intermediates, including final BAMs (most aggressive).
  - ``all``: keeps everything (most disk usage).
  - ``custom``: uses the per-category switches ``keep_trimmed_fastqs``,
    ``keep_final_bams``, ``keep_merged_bams``, ``keep_shifted_bams`` (ATAC
    Tn5-shifted BAMs), ``keep_cx_reports`` (per-replicate CX methylation
    reports; keep them if you pass per-replicate data to DMRcaller), and
    ``keep_dmc_intermediates`` (dmC aligned modBAMs and bedMethyl pileups).

- ``use_node_tmpdir``: Default ``false``. See
  :ref:`TMPDIR routing <ref-tmpdir-routing>`.

Intermediate input formats
^^^^^^^^^^^^^^^^^^^^^^^^^^

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
^^^^^^^^^^^^^^^^^^^^^^^

Set via ``chip_mapping_strategy`` in the options file or
``--chip-mapping-strategy`` (``atac_mapping_strategy`` /
``--atac-mapping-strategy`` for ATAC-seq):

- ``default``: Standard mapping parameters
- ``repeat``: Centromere-specific mapping (more sensitive; forces Bowtie2 regardless of ``chip_aligner``)
- ``repeatall``: Centromere mapping with relaxed MAPQ (forces Bowtie2)
- ``all``: Relaxed mapping parameters

DMRs parameters
^^^^^^^^^^^^^^^

DNA methylation data is analyzed per context. The active contexts are
controlled by ``methylation_contexts`` (default: ``["CG", "CHG", "CHH"]``, or
``--methylation-contexts CG,CHG,CHH``). This option gates per-context
bigwigs, DMR calls, and PCA plots.

For animal genomes where non-CpG methylation is negligible, set:

.. code-block:: yaml

   methylation_contexts: ["CG"]

to skip empty CHG/CHH outputs. Subcontexts (CAG, CAA, etc.) are not yet
supported.

Two DMR callers are available for Bismark-aligned samples, selected with
``dmr_caller`` (or ``--dmr-caller``):

- ``metilene`` (default): C++ binary segmentation, multi-threaded and
  significantly faster than DMRcaller on high-coverage replicate data.
- ``dmrcaller``: R/Bioconductor
  `DMRcaller <https://www.bioconductor.org/packages/release/bioc/html/DMRcaller.html>`__
  (DOI: 10.18129/B9.bioc.DMRcaller), using ``computeDMRsReplicates`` with beta
  regression and replicate variance. Slower on real data; use it when you want
  the beta-regression model. Pair-context jobs run serially (one chromosome
  at a time), so high-coverage multi-replicate pairs can take 1-4 hours.

dmC samples use ``dmc_dmr_method`` (default ``"dmrcaller"``; ``"modkit"`` is
experimental and only valid for dmC versus dmC comparisons) regardless of
``dmr_caller``.

Thresholds are set in the ``dmr_thresholds:`` block. Defaults are tuned for
Arabidopsis leaves (`Catoni et al. 2018
<https://academic.oup.com/nar/article/46/19/e114/5050634>`__); methylation
varies widely across species and tissues (e.g. maize CHH is very low; pollen
differs from leaf), so adjust as needed. Omit a key to use its default.

.. code-block:: yaml

   dmr_thresholds:
     min_diff:       {CG: 0.3, CHG: 0.2, CHH: 0.1}  # min mean methylation difference
     sigma_n:        3.0    # multiplier for min_diff: "auto"
     min_cytosines:  5      # min CpG/cytosine count per DMR window (both callers)
     q_value:        0.01   # FDR q-value cutoff (both callers)
     min_size:       50     # min DMR length in bp (both callers)
     bin_size:       200    # DMRcaller: window size (bp)
     min_gap:        200    # DMRcaller: min gap between DMRs (bp)
     min_reads:      3      # DMRcaller: min reads per cytosine
     maxdist:        300    # metilene: max distance between CpGs in a segment, nt (-M)
     valley:         {CG: 0.7, CHG: 0.7, CHH: 0.3}  # metilene: valley filter (-v)
     maxseg:         {CG: -1, CHG: -1, CHH: 10000}  # metilene: max segment cap in CpGs (-G; -1 = no cap)

``min_diff``, ``min_cytosines``, ``q_value`` and ``min_size`` apply to both
callers; ``bin_size``, ``min_gap`` and ``min_reads`` are DMRcaller-only, and
``maxdist``, ``valley`` and ``maxseg`` are metilene-only. Each caller ignores
the other's keys. ``q_value`` is applied to DMRcaller's BH-adjusted p-value and
to metilene's q-value column. For metilene, ``min_size`` is applied after
calling as ``(end - start) >= min_size``.

To estimate ``min_diff`` from the data instead, set ``min_diff: "auto"``
(string). For each comparison pair and context, a calibration step first
computes ``sigma_n`` x σ, where σ is the standard deviation of the
per-position between-group methylation difference. The threshold scales with
baseline methylation (CG is highly methylated, so a higher threshold; CHH is
near zero, so a lower one) and is floored at 0.05. Tune ``sigma_n`` (default
3.0) to tighten or relax sensitivity. ``auto`` is recommended outside
Arabidopsis leaf tissue.

Each threshold has a command-line override:

.. code-block:: bash

   epicc run --samples my_samples.tsv --dmr-caller metilene \
     --dmr-min-diff CG=0.3,CHG=0.2,CHH=0.1 --dmr-min-cytosines 5 --dmr-q-value 0.01

``--dmr-min-diff`` accepts ``auto``, a single value applied to all contexts
(e.g. ``0.2``), or per-context values. ``--dmr-sigma-n`` sets ``sigma_n``.

A custom parameter sweep mode is available. Toggle ``custom_script_dmrs: true``
in the options file to use it (``workflow/scripts/R_call_DMRs_custom.R``; slower
and more resource intensive).

Peak calling
^^^^^^^^^^^^

ChIP peaks are called with MACS2; ``chip_callpeaks.params`` (default
``--keep-dup all --nomodel``, or ``--chip-callpeaks-params``) sets its
parameters, and ``chip_callpeaks.peaktype`` maps IP targets to a default peak
type. CUT&RUN and CUT&Tag use peak-shape-aware defaults: epic2 for ``broad``
and SEACR for ``narrow``, with MACS2 as a fallback for either. Choose with
``cut_callpeaks.broad_caller`` (``epic2``, ``seacr``, ``macs2``) and
``cut_callpeaks.narrow_caller`` (``seacr``, ``macs2``), or
``--cut-broad-caller`` / ``--cut-narrow-caller``. ``cut_callpeaks`` also holds
the per-caller parameters (``epic2_params``, ``seacr_norm``,
``seacr_threshold``, ``macs2_params_pe``, ``macs2_params_se``); set
``seacr_norm`` to ``"non"`` when an IgG control is supplied.

In-line customization
^^^^^^^^^^^^^^^^^^^^^

.. _ref-named-overrides:

Commonly tuned options have named ``epicc run`` / ``epicc validate`` flags that
override the options file: ``--chip-aligner``, ``--atac-aligner``,
``--chip-mapping-strategy``, ``--atac-mapping-strategy``,
``--[no-]full-analysis``, ``--[no-]te-analysis``, ``--[no-]go``,
``--[no-]rna-deduplicate``, ``--methylation-contexts``, ``--dmr-caller``,
``--dmr-min-diff``, ``--dmr-min-cytosines``, ``--dmr-q-value``,
``--dmr-sigma-n``, ``--chip-callpeaks-params``, ``--cut-broad-caller`` and
``--cut-narrow-caller``. Nested settings (``dmr_thresholds.*``,
``chip_callpeaks.params``, ``cut_callpeaks.*``) are applied through a temporary
merged config file, so sibling keys not given on the command line keep the
values from the options file.

Any other configuration option can be overridden by passing it after ``--`` to
``epicc run``, using Snakemake's ``--config`` flag:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv -- --config chip_mapping_strategy="repeatall"
