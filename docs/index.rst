PANGEApy Documentation
======================

**PANGEApy** is a Python package for automated cell type and metadata annotation
using the large-scale PANGEA reference atlas built across human organs.

It provides a unified interface for:

- **Cell-level annotation** using the ``CellAnnotator`` module
- **Metadata-level annotation** (organ, phenotype) using the ``MetaAnnotator`` module
- **Uncertainty estimation** to detect missing or novel cell types

----

Interactive Tools
-----------------

Explore your annotation results directly in the browser — no installation required.

.. list-table::
   :widths: 30 70
   :header-rows: 0

   * - `Cell Annotation Explorer <cellplot/>`_
     - UMAP visualization of cell type annotations (Level1 / Level2 / Combined)
   * - `Meta Annotation Plot <metaplot/>`_
     - Organ and phenotype distribution bar charts with clinical metadata grouping
   * - `Uncertainty Explorer <uncertainty/>`_
     - Identify missing or novel cell types via uncertainty score visualization

----

.. toctree::
   :maxdepth: 1
   :caption: Tutorials

   01_vignette_cell_annotation
   02_vignette_meta_annotation
   03_vignette_identifying_missing_cells
   04_vignette_plus_one_model

.. toctree::
   :maxdepth: 1
   :caption: API Reference

   api/cell
   api/meta
   api/models
