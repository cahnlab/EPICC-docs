=====
Usage
=====


Running the pipeline
====================

The main entry point is the ``epicc`` CLI wrapper at the repository root. It
is a thin argparse wrapper around Snakemake that handles configuration
validation, placeholder detection, and TMPDIR routing automatically. The
subcommands are ``run``, ``validate``, ``output``, ``unlock``, ``perf``,
``clean`` and ``init-profile``; run ``epicc --help`` or
``epicc <subcommand> --help`` for the full option list. Raw ``snakemake ...``
invocations still work but bypass these checks.

1. To run the pipeline locally:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv --cores 12

2. To run the pipeline on a SLURM cluster:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv --profile profiles/slurm

Launch cluster runs from a login or development node, not from inside an
``sbatch`` job (see :ref:`Resources and Profiles <ref-resources-profiles>`).
To reduce terminal output, redirect to a log file and run in the background:

.. code-block:: bash

   epicc run --samples config/my_samples.tsv --profile profiles/slurm > epicc.log 2>&1 &

3. To run the pipeline on a UGE cluster (using qsub):

.. code-block:: bash

   epicc run --samples config/my_samples.tsv --profile profiles/uge

If using a profile, review and adapt the profile configuration to your cluster
before running. ``epicc init-profile slurm NAME`` (or ``uge``) copies a bundled
template to ``~/.config/snakemake/NAME/config.yaml``, after which
``--profile NAME`` selects it; this is the recommended setup for conda-installed
copies and for sharing a profile across projects. Use ``--force`` to overwrite
an existing profile.

When neither ``--profile`` nor ``--cores`` is given, ``epicc run`` uses
``profiles/slurm`` if ``sbatch`` is available and otherwise runs locally on half
of the available cores.

4. Key ``epicc run`` flags:

   - ``--options FILE`` — path to the options YAML file (default: ``config/epicc-options.yaml``)
   - ``--samples FILE`` — path to the sample sheet TSV (overrides the value in the options file)
   - ``--cores N`` — number of cores for local execution (default: half of available CPUs)
   - ``--profile DIR`` — cluster (execution) profile directory or name
   - ``--workflow-profile DIR`` — workflow profile with per-rule resources and threads (default: ``profiles/default``)
   - ``--output-dir DIR`` — results directory prefix (default: ``results``)
   - ``--genome-dir DIR`` — genome directory prefix (default: ``genomes``)
   - ``--keep-intermediates TIER`` — intermediate file retention: ``none``, ``standard`` (default), ``custom``, ``all``
   - ``--use-node-tmpdir`` — skip the workflow's TMPDIR override and inherit the cluster's default (see :ref:`TMPDIR routing <ref-tmpdir-routing>`)
   - ``--target TARGET`` — named target (default: ``all``), e.g. ``map_only``
   - ``--until RULE`` — stop after this rule completes
   - ``--forcerun RULE [RULE ...]`` — force re-execution of specific rules
   - ``--auto-unlock`` — if a stale Snakemake lock is found at startup, clear it and continue. By default the run halts and reports the lock, like Snakemake. Use only when no other run is active in the directory; ``epicc unlock`` clears a lock manually.
   - ``--no-rerun-incomplete`` — skip re-running jobs flagged incomplete by a prior interrupted run
   - ``--verbose`` / ``-v``, ``--quiet`` / ``-q`` — print shell commands, or keep output minimal
   - ``-- SNAKEMAKE_ARGS`` — anything after ``--`` is forwarded verbatim to Snakemake

5. Named overrides for commonly tuned options. These take precedence over the
   options file and are accepted by both ``epicc run`` and ``epicc validate``:

   - ``--chip-aligner {bowtie2,chromap}``, ``--atac-aligner {bowtie2,chromap}``
   - ``--chip-mapping-strategy STRATEGY``, ``--atac-mapping-strategy STRATEGY`` (``default``, ``repeat``, ``repeatall``, ``all``)
   - ``--full-analysis`` / ``--no-full-analysis``
   - ``--te-analysis`` / ``--no-te-analysis``
   - ``--go`` / ``--no-go``
   - ``--rna-deduplicate`` / ``--no-rna-deduplicate``
   - ``--methylation-contexts CONTEXTS`` — comma-separated, e.g. ``CG`` (default ``CG,CHG,CHH``)
   - ``--dmr-caller {metilene,dmrcaller}``
   - ``--dmr-min-diff SPEC`` — ``auto``, a single float for all contexts (e.g. ``0.2``), or ``CG=0.3,CHG=0.2,CHH=0.1``
   - ``--dmr-min-cytosines N``, ``--dmr-q-value Q``, ``--dmr-sigma-n N``
   - ``--chip-callpeaks-params STR`` — MACS2 parameters for ChIP peak calling
   - ``--cut-broad-caller {epic2,seacr,macs2}``, ``--cut-narrow-caller {seacr,macs2}``

   See :ref:`Configuration <ref-named-overrides>` for what each setting does.

6. Validate configuration and optionally generate a workflow graph before running:

.. code-block:: bash

   epicc validate --samples config/my_samples.tsv
   epicc validate --samples config/my_samples.tsv --dag dag.png
   epicc validate --samples config/my_samples.tsv --rulegraph rules.png

``epicc validate`` checks the options file and sample sheet, then runs a
Snakemake dry-run. ``--dag`` and ``--rulegraph`` write the job-level or
rule-level graph (format from the file extension; ``png``/``svg``/``pdf``
need Graphviz ``dot``).

7. Other subcommands:

   - ``epicc unlock`` — remove Snakemake's directory lock after a failed or interrupted run (equivalent to ``snakemake --unlock``).
   - ``epicc perf [LOGFILE]`` — timing report (per-rule timing, slowest jobs, parallelism) from Snakemake logs. By default all logs from the most recent resumed run are aggregated; pass a ``.snakemake.log`` file or a run's results directory, ``--single`` for the newest log only, or ``--html FILE`` for an HTML report with a Gantt timeline.
   - ``epicc clean`` — remove pipeline artifacts: ``--intermediates`` (checkpoint files, forcing reanalysis on the next run), ``--conda-envs`` (unused conda environments), ``--logs`` (old Snakemake logs, keeping the most recent), ``--tmp`` (see :ref:`TMPDIR routing <ref-tmpdir-routing>`), or ``--all``.
   - ``epicc output`` — generate a single targeted output (see `Additional Output Options`_).

For full Snakemake documentation: https://snakemake.readthedocs.io/en/stable/


Conda environment maintenance
==============================

Rule-specific conda environments are defined under ``workflow/envs/`` and are
created automatically on first use by Snakemake. To clean up orphaned
environment directories that accumulate under ``.snakemake/conda/`` after
pipeline upgrades:

.. code-block:: bash

   snakemake --sdm conda --conda-cleanup-envs

The shipped SLURM profile sets ``conda-cleanup-pkgs: tarballs``, which frees
downloaded package archives after each environment is built without removing
the installed environment.

.. _ref-build-envs:

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
the failing creation step. If you do this, launch the run itself from the login
node too, rather than from inside a job (see
:ref:`Resources and Profiles <ref-resources-profiles>`).


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

To disable this override, either:

- Pass ``--use-node-tmpdir`` on the command line, or
- Set ``use_node_tmpdir: true`` in ``config/epicc-options.yaml``

This does **not** mean "use ``/tmp``": EPICC then sets no ``TMPDIR`` at all and
jobs inherit one from the environment, in this order of precedence:

1. ``TMPDIR`` exported in the shell you launch from (it reaches the jobs, since
   the SLURM plugin submits with ``sbatch --export=ALL``)
2. ``TMPDIR`` set by the site: a SLURM prolog, ``job_container/tmpfs``, or an
   environment module
3. ``TMPDIR`` set by the SLURM profile's ``precommand``, which takes effect only
   in this mode because nothing overwrites it afterwards
4. Each tool's own default, usually ``/tmp``, only if nothing above set one

So directing temporary files at node-local scratch, or at a filesystem under
less metadata pressure, is just ``export TMPDIR=...`` before the run.

Two rules opt out in either mode: ``STAR_map_pe``/``STAR_map_se`` keep temporary
files in the ``genomes/`` and ``results/`` trees, and Bismark uses its own
``--temp_dir`` under ``results/mC/mapped/<sample>/``. The routing therefore
trades cluster ``/tmp`` pressure for metadata load on the project filesystem
(one directory created and removed per job). If your filesystem administrators
report metadata contention, ``use_node_tmpdir: true`` and a lower ``jobs:``
setting in the profile reduce EPICC's contribution.

Each scratch directory is removed by an ``EXIT`` trap when its job finishes,
including on most failures, but ``SIGKILL`` (a SLURM timeout, an OOM kill, or a
node failure) bypasses the trap and leaks the directory. Reclaim leaked
directories with:

.. code-block:: bash

   epicc clean --tmp

A directory is removed only if it does not belong to a SLURM job that is still
queued or running and it has been untouched for ``--tmp-min-age`` hours
(default 6). Directories left alone are listed with the reason. If ``squeue``
is unavailable, SLURM-named directories are skipped rather than assumed dead.


Intermediate Target Rules
=========================

Two named intermediate targets are available when only part of the analysis is
needed:

- ``map_only`` — performs alignment only for all samples; returns BAM files,
  QC files, and mapping metrics.

  .. code-block:: bash

     epicc run --samples config/my_samples.tsv --target map_only

- ``coverage_chip`` — creates bigwig coverage tracks for all ChIP, CUT&RUN and
  CUT&Tag samples. Bin size defaults to 1 bp (configurable via
  ``chip_tracks: binsize:`` in the options file).

  .. code-block:: bash

     epicc run --samples config/my_samples.tsv --target coverage_chip


Additional Output Options
=========================

The following outputs can be generated once the whole pipeline has run at least
once. Use ``epicc output --plot-type TYPE --input-file PATH [options]`` to
generate them. ``--plot-type`` is one of ``rnaseq-histogram``, ``go``,
``motifs``, ``srna-clusters``, ``heatmap``, ``metaplot``, ``browser``;
``--plot-label`` names the output files (default: the input file's basename).
``--analysis-name NAME`` overrides ``analysis_name`` and ``--dry-run`` prints the
Snakemake plan without running.


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
``gaf_file`` and ``gene_info_file`` under the corresponding genome block (or
pass ``--go``). See
:ref:`Help GO <ref-help-go-analysis>` for database construction details.

.. code-block:: bash

   epicc output --plot-type go \
     --input-file data/target_genes.txt \
     --plot-label my_genes_of_interest \
     --ref-genome ColCEN

By default the enrichment background is all genes in the genome. To restrict it
(e.g. to the genes expressed in your experiment), pass a tab-delimited file
whose first column is gene IDs matching the reference GFF with
``--background``; this sets ``rnaseq_background_file`` for this run:

.. code-block:: bash

   epicc output --plot-type go \
     --input-file data/target_genes.txt \
     --plot-label my_genes_of_interest \
     --ref-genome ColCEN \
     --background data/expressed_genes.txt

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

If the BED file is stranded, heatmaps use the sense-strand track for properly
stranded data types (RNAseq, sRNA), so each region contributes one column per
sample rather than separate plus and minus columns. Regions whose strand column
is neither ``+`` nor ``-`` (GFF3 ``?``, or ``.``) have no sense strand and are
plotted as their own region group; a TE annotation with a handful of these is
still treated as stranded. Disable with ``stranded_heatmaps: false`` in the
options file.

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

Alternatively, ``epicc clean --intermediates`` removes all checkpoint files so
the next run repeats the analyses.

Several ``epicc output`` invocations can be chained in a script to iterate over
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
