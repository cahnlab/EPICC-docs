Installation
------------

1. Clone the repository:

.. code-block:: bash

   git clone https://github.com/cahnlab/EPICC.git

Or via SSH:

.. code-block:: bash

   git clone git@github.com:cahnlab/EPICC.git

Then enter the directory:

.. code-block:: bash

   cd EPICC

2. Install Snakemake and other required packages from the dependency file and activate the environment:

.. code-block:: bash

   conda create -n epicc -y --file config/epicc-env.txt
   conda activate epicc

If you do not have conda yet, follow the directions `on the conda website linked here`_.

.. _on the conda website linked here: https://docs.conda.io/projects/conda/en/stable/user-guide/install/

``config/epicc-env.txt`` installs Snakemake (9.0 or later) and the SLURM and
generic-cluster executor plugins (the latter is used by the UGE profile). The
SLURM jobstep plugin is pinned below 0.6.1; see :ref:`Resources and Profiles
<ref-resources-profiles>`. If you want to run the pipeline on another platform,
also install the corresponding Snakemake executor plugin (`executor plugin options here`_).

.. _executor plugin options here: https://snakemake.github.io/snakemake-plugin-catalog/index.html

3. Check the installation and your inputs. ``epicc validate`` checks the options
file and sample sheet, then runs a Snakemake dry-run:

.. code-block:: bash

   epicc validate --samples your_samples.tsv

Rule-specific conda environments (under ``workflow/envs/``) are created
automatically on first use. On a cluster, set up a scheduler profile with
``epicc init-profile`` first; see :ref:`Resources and Profiles
<ref-resources-profiles>`.
