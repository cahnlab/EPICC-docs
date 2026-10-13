=====
Usage
=====


Running the pipeline
====================

The main entry point is the ``epicc`` CLI wrapper at the repository root. It
handles configuration validation, placeholder detection, and TMPDIR routing
automatically. Raw ``snakemake ...`` invocations still work but bypass these
checks.

1. To run the pipeline locally:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv --cores 12

2. To run the pipeline on a SLURM cluster:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv --profile profiles/slurm

To reduce terminal output, redirect to a log file and run in the background:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv --profile profiles/slurm > epicc.log 2>&1 &

3. To run the pipeline on a UGE cluster (using qsub):

.. code-block:: bash

   epicc run --samples config/my_samples.tsv --profile profiles/uge

If using a profile, review and adapt the profile configuration to your cluster before running.

4. Key ``epicc run`` flags:

   - ``--options FILE`` — path to the options YAML file (default: ``config/epicc-options.yaml``)
   - ``--samples FILE`` — path to the sample sheet TSV (overrides the value in the options file)
   - ``--cores N`` — number of cores for local execution (default: half of available CPUs)
   - ``--profile DIR`` — cluster execution profile directory
   - ``--output-dir DIR`` — results directory prefix (default: ``results``)
   - ``--genome-dir DIR`` — genome directory prefix (default: ``genomes``)
   - ``--keep-intermediates TIER`` — intermediate file retention: ``none``, ``standard`` (default), ``custom``, ``all``
   - ``--use-node-tmpdir`` — skip the workflow's TMPDIR override and inherit the cluster's default (see :ref:`TMPDIR routing <ref-tmpdir-routing>`)
   - ``--no-rerun-incomplete`` — skip re-running jobs flagged incomplete by a prior interrupted run
   - ``--forcerun RULE [RULE ...]`` — force re-execution of specific rules
   - ``-- SNAKEMAKE_ARGS`` — anything after ``--`` is forwarded verbatim to Snakemake

5. Validate configuration and optionally generate a workflow graph before running:

.. code-block:: bash

   epicc validate --samples config/my_samples.tsv
   epicc validate --samples config/my_samples.tsv --dag dag.png
   epicc validate --samples config/my_samples.tsv --rulegraph rules.png

For a full list of subcommands (``run``, ``validate``, ``output``, ``unlock``, ``perf``, ``clean``) run
``epicc --help`` or ``epicc <subcommand> --help``.

For full Snakemake documentation: https://snakemake.readthedocs.io/en/stable/


Conda environment maintenance
==============================

Rule-specific conda environments are defined under ``workflow/envs/`` and are
created automatically on first use by Snakemake. To clean up orphaned
environment directories that accumulate under ``.snakemake/conda/`` after
pipeline upgrades:

.. code-block:: bash

   snakemake --sdm conda --conda-cleanup-envs

Pre-building environments before sbatch-wrapped runs
-----------------------------------------------------

On some clusters, ``conda env create --prefix`` fails when invoked from inside
a SLURM job allocation while the same command works from a login node. If you
launch ``epicc run`` via ``sbatch``, build all rule environments once from a
login or development node first:

.. code-block:: bash

   epicc validate --build-envs --samples your_samples.tsv

This runs standard configuration checks and a dry-run, then calls
``snakemake --conda-create-envs-only`` to populate ``.snakemake/conda/``.
Subsequent sbatch-wrapped runs will reuse the pre-built environments and skip
the failing creation step.


.. _ref-tmpdir-routing:

TMPDIR routing
==============

By default, every pipeline job sets ``TMPDIR`` to a per-job subdirectory under
``{output_dir}/.tmp/`` (e.g. ``results/.tmp/<SLURM_JOB_ID>.<PID>``). Tools that
spill large temporary data through ``TMPDIR`` — such as ``samtools sort``,
STAR, ``fasterq-dump``, and deeptools — therefore write to the project
filesystem rather than the cluster's ``/tmp``. This avoids ``ENOSPC`` errors
on sites where ``/tmp`` is a tmpfs sized to the job's RAM allocation (e.g.
SLURM ``JobContainerType=job_container/tmpfs``).

To disable this override and inherit whatever ``TMPDIR`` the cluster provides
(e.g. when ``/tmp`` is fast local NVMe scratch with adequate capacity), either:

- Pass ``--use-node-tmpdir`` on the command line, or
- Set ``use_node_tmpdir: true`` in ``config/epicc-options.yaml``

Bismark's temporary files are directed to ``results/mC/mapped/<sample>/``
via its own ``--temp_dir`` flag and are unaffected by either setting.


Intermediate Target Rules
=========================

Two named intermediate targets are available when only part of the analysis is
needed:

- ``map_only`` — performs alignment only for all samples; returns BAM files,
  QC files, and mapping metrics.

  .. code-block:: bash

     epicc run --samples config/my_samples.tsv -- map_only

- ``coverage_chip`` — creates bigwig coverage tracks for all ChIP and ATAC
  samples. Bin size defaults to 1 bp (configurable via ``chip_tracks: binsize:``
  in the options file).

  .. code-block:: bash

     epicc run --samples config/my_samples.tsv -- coverage_chip


Additional Output Options
=========================

The following outputs can be generated once the whole pipeline has run at least
once. Use ``epicc output --plot-type TYPE --input-file PATH [options]`` to
generate them.


Plotting RNAseq expression levels on target genes
-------------------------------------------------

Given a list of genes (and optional labels), plots expression levels across all
samples in the analysis. Genes uniquely differentially regulated in one sample
versus others are color coded. Requires the RData file created during the
differential expression analysis.

Provide a target gene list file (one column of gene IDs matching the GTF of the
reference genome; an optional second column provides gene labels) and run:

.. code-block:: bash

   epicc output --plot-type rnaseq-histogram \
     --input-file data/target_genes.txt \
     --plot-label my_genes_of_interest \
     --ref-genome TAIR10

Output is a single PDF file:
``results/RNA/plots/plot_expression__<analysis_name>__<ref_genome>__<label>.pdf``

The separator between variables is two underscores ``__``.

See :ref:`Example output <fig-rna-exp-level>`.

.. _ref-new-go-analysis:

Performing GO analysis on target genes
--------------------------------------

Given a file containing a list of genes, performs Gene Ontology enrichment
analysis (optionally against a custom background; default is all genes in the
reference genome).

By default, GO analysis is not performed because it requires manually building
a database. To activate it, set ``GO: true`` in the options file and define
``gaf_file`` and ``gene_info_file`` under the corresponding genome block. See
:ref:`Help GO <ref-help-go-analysis>` for database construction details.

.. code-block:: bash

   epicc output --plot-type go \
     --input-file data/target_genes.txt \
     --plot-label my_genes_of_interest \
     --ref-genome ColCEN

Output includes two PDF treemaps under ``results/RNA/plots/``:
``topGO_<label>_BP_treemap.pdf`` (biological process) and
``topGO_<label>_MF_treemap.pdf`` (molecular function), plus corresponding
enrichment tables under ``results/RNA/GO/``.

See :ref:`Example output <ref-go-analysis>`.


Finding motifs on target regions
--------------------------------

Given a BED file of regions, performs motif analysis with the MEME suite.

By default motifs analysis is only performed on the final selected TF peak
files (``motifs: true`` in the options file). To also run on all replicates
and pairwise IDR peaks, set ``motifs_allreps: true``. A plant motifs database
is used by default for TOMTOM. Download the appropriate file from JASPAR and
update ``jaspar_db`` and ``motif_ref_genome`` in the options file.

.. code-block:: bash

   epicc output --plot-type motifs \
     --input-file data/target_peaks.bed \
     --plot-label my_regions_of_interest \
     --ref-genome ColCEN

Output is the folder ``results/ChIP/<label>`` containing a ``meme``
subdirectory and optionally a ``tomtom`` subdirectory, as described in the
`MEME suite documentation <https://meme-suite.org/meme/index.html>`__.

For regions over 500 bp, only the middle 400 bp will be used.

See :ref:`Example output <ref-motifs-analysis>`.


Performing sRNA differential analysis on target regions
-------------------------------------------------------

Given a BED or GFF file, runs ShortStack followed by differential expression
with edgeR, limiting mapping and counts to the provided loci.

.. code-block:: bash

   epicc output --plot-type srna-clusters \
     --input-file data/miRNA.gff \
     --plot-label miRNAs \
     --ref-genome ColCEN

Output: ``results/sRNA/clusters/<analysis_name>__<ref_genome>__on_<label>/``

The BED or GFF file **must have** a column called "Name" (the 4th column of a
BED file or the 9th column of a GFF3).

See :ref:`Example output <ref-cluster-diff-expression>`.


Plotting heatmap on target regions
----------------------------------

Given a BED file, plots a heatmap using deeptools. Edit ``heatmap_target_file``
and ``heatmap_target_file_label`` in the options file, or pass them via the
``--input-file`` and ``--plot-label`` flags.

.. code-block:: bash

   epicc output --plot-type heatmap \
     --input-file data/target_genes.bed \
     --plot-label interesting_genes \
     --ref-genome ColCEN \
     --matrix regions \
     --env most

- ``--matrix`` can be ``regions`` (scaled features), ``tss`` (reference point
  on TSS), or ``tes`` (reference point on TES).
- ``--env`` selects the data types: ``most`` includes all data types except mC;
  use ``mC``, ``mCG``, ``mCHG``, or ``mCHH`` for methylation-specific heatmaps;
  other choices are ``ChIP``, ``ATAC``, ``RNA``, ``sRNA``.

Since mC requires different deeptools parameters, it is handled independently.
To produce a mC heatmap sorted by the same region order as the other samples,
use the ``Heatmap_sorted__`` Snakemake target directly.

Output is a PDF file, or two if a sorted mC heatmap was generated.

By default, heatmaps are scaled by data type (``heatmaps_scales: "type"``
in the options file). Change to ``"default"`` for a single scale or
``"sample"`` for per-sample scaling.

By default, regions are sorted by mean signal (``heatmaps_sort_options: "mean"``).
Change to ``"median"`` or ``"no"`` to preserve the BED file order.

If the BED file is stranded, heatmaps will split plus/minus strand for
properly stranded data types (RNAseq, sRNA). Disable with
``stranded_heatmaps: false`` in the options file.

Color scheme defaults: ``seismic`` for all non-mC samples, ``Oranges`` for mC.
Change via ``heatmaps_plot_params`` in the options file.

Window sizes (``before``, ``after``, scaled-region ``middle``) and bin size
(``binsize``) are configurable in ``heatmaps`` per matrix type in the options
file.

See :ref:`Example output <fig-heatmap-deeptools>`.


Plotting metaplot profiles on target regions
--------------------------------------------

Given a BED file, plots a metaplot profile using deeptools. Uses the same
``heatmap_target_file`` / ``heatmap_target_file_label`` options file entries as
the heatmap above.

.. code-block:: bash

   epicc output --plot-type metaplot \
     --input-file data/target_genes.bed \
     --plot-label interesting_genes \
     --ref-genome ColCEN \
     --matrix regions \
     --env all

``--matrix`` and ``--env`` options are identical to the heatmap command above.
Use ``--env all`` to include all samples including mC.

Output is two PDF files where the samples are grouped by regions or not.

By default, profiles are scaled by data type (``heatmaps_scales: "type"``).
By default, profiles represent the mean across all regions
(``profiles_scale: "mean"``; change to ``"median"``).
Plot type defaults to ``"lines"``; see deeptools documentation for other
options and edit ``profiles_plot_params`` in the options file.

Window sizes and bin size are configured in ``heatmaps`` per matrix type in
the options file (same settings as heatmap).

See :ref:`Example output <fig-metaplot-deeptools>`.


Plotting browser screenshots on target regions
----------------------------------------------

Given a region file, plots browser screenshots using R packages. Edit
``browser_target_file`` and ``browser_target_file_label`` in the options file.

.. code-block:: bash

   epicc output --plot-type browser \
     --input-file data/target_loci.bed \
     --plot-label target_loci \
     --ref-genome ColCEN \
     --env all

The target file is a BED-like file with the following columns:

+------+-------+------+-------+-----------------+----------------------------+----------------------------+
| Chr  | Start | End  | ID    | Binsize (in bp) | Highlight_starts (optional)| Highlight_widths (optional)|
+======+=======+======+=======+=================+============================+============================+
| chr1 | 1000  | 5000 | Peak1 | 1               | 3000,4000                  | 50,200                     |
+------+-------+------+-------+-----------------+----------------------------+----------------------------+

Each region is printed individually and merged into a final PDF.

Highlight columns are optional; they mark regions of the browser view with
colored boxes. As many highlights as needed can be provided as comma-separated
lists — the first highlight is blue and all others are red. Positions in
``Highlight_starts`` are absolute chromosome coordinates (not relative to the
region ``Start``). For example, for a region ``chr1:1000-5000``, ``col6=3000,4000``
and ``col7=50,200`` produces a blue box at ``chr1:3000-3050`` and a red box at
``chr1:4000-4200``.

Use ``--env all`` to include all samples, ``most`` for all data types except mC,
or any single environment (``ChIP``, ``ATAC``, ``RNA``, ``sRNA``, ``mC``).

By default, no TE file is used. To add TE annotations, supply a BED file via
``browser_TE_file`` in the options file.

See :ref:`Example output <fig-browser-plot>`.


Rerunning a specific analysis
-----------------------------

Changing parameters in the options file or sample sheet will trigger a rerun of
affected outputs. To force Snakemake to recreate a specific target:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv -- results/combined/plots/srna_sizes_stats_myanalysis_sRNA.pdf --force

If only the combined analysis needs to be rerun (not data-type-specific steps),
delete the checkpoint files in ``results/combined/chkpts/`` and in the relevant
environment directory ``results/<env>/chkpts/<env>_analysis__<analysis_name>__<ref_genome>.done``.

Several ``epicc run`` invocations can be chained in a script to iterate over
parameter values:

.. code-block:: bash

   for scale in default sample type; do
     epicc output --plot-type heatmap \
       --input-file data/target_genes.bed \
       --plot-label interesting_genes_${scale} \
       --ref-genome ColCEN \
       --matrix regions \
       --env most
   done
