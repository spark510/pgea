PANGEApy Documentation
======================

**PANGEApy** is a Python package for automated cell type and metadata annotation
using the large-scale PANGEA reference atlas built across human organs.

It provides a unified interface for:

- **Cell-level annotation** using the ``CellAnnotator`` module
- **Metadata-level annotation** (organ, phenotype) using the ``MetaAnnotator`` module
- **Uncertainty estimation** to detect missing or novel cell types

----

.. toctree::
   :maxdepth: 1
   :caption: Interactive Tools

   tool_cellplot
   tool_metaplot
   tool_uncertainty

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
