..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Many algorithms produce scalar metric values describing data quality. Storing those efficiently in butler is extremely important and in some cases being able to include the metrics in graph building is also required. This technote will explore how metrics can be handled efficiently.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

Pipelines often generate scalar quantities as part of processing.
These metrics, for example those from ``faro`` :cite:`DMTN-211`, can include items such as the time an algorithm takes to execute, the number of sources in the image, or assessments of image quality.
The metrics are, currently, treated as datasets by the butler in just the same way as source catalogs or processed images:
A ``lsst.verify.Measurement`` object is created containing the metric, it is stored in the butler with a ``butler.put()``, and it is then serialized to disk and stored in the datastore, currently in YAML format. [*]_

.. [*] Storing the metrics in their native JSON format would make reading them much quicker but does not solve the large scale problem when tens of thousands need to be read by a user.

From a pipeline execution perspective having tasks generate individual metrics is the right solution since it allows pipelines to be tuned to only generate the metrics that are required.
The problem, though, is that with a large data release there are tens of thousands of metrics created and each one is a tiny file in an object store.
This makes it almost impossible for the verification and validation team to read more than a few hundred into an analysis notebook.

Butler Datasets
===============

Every metric generated in a pipeline is treated as a butler dataset.
A butler dataset has a UUID that can uniquely identify it, but it also is guaranteed to be locatable using:

* A RUN collection.
* A dataset type.
* A data coordinate.

A metrics database that contains datasets from a butler repository therefore needs to include the run and data coordinate as well as the dataset type.
The data coordinate allows the metric to be located in time or space, and the time-based lookup can be used to associate the metric with other EFD telemetry.

Furthermore, butler has the concept of CHAINED collections where a dataset can be located by returning the first match in the chain.
How CHAINED collections map to a metrics database outside of butler control is not yet determined.

Ways Forward
============

There are some ways by which the metrics tracking system can be improved.

* Write collation pipeline tasks that will run in the batch system.
  These tasks will collect related metrics and store them in a single butler dataset, possibly in parquet format.
* Transfer metrics into the SQuaSH system :cite:`SQR-009` as an afterburner (possibly by reading the collated parquet files).
* Create a Sasquatch/SQuaSH :cite:`SQR-068` datastore backend that can be used directly from batch workflows to stream metrics into the EFD (whilst also storing them as individual datasets using a chained datastore).

Sasquatch
=========

Sasquatch :cite:`SQR-068` is the next generation of the SQuaSH and EFD systems, a unified service to store and analyze telemetry, events, and metrics for Rubin Observatory.
The current design allows for metrics to be loaded that can track the dataset type, processing run, and dataId.
Sasquatch is based on InfluxDB, a time-series database that has been used by the Data Management team to track how metrics evolve with changes to software versions.
The current design allows for metrics to be loaded with associated metadata such as dataset type, processing run, and dataId.

In the context of an operational observatory, it will be necessary for metrics to be associated with the time of observation so that it will be possible to correlate telemetry from the EFD with metrics (for example plotting measured seeing against wind speed).

Each (exposure-based) metric stored in the butler repository will have two useful timestamps.
The first is the timestamp of the original observation and the second is the time the metric was calculated.
Additionally, the software version and other provenance information can be useful.

Sasquatch should not be responsible for finding the time of observation for a visit or exposure, this timestamp will be obtained by the code that sends the Butler metrics to Sasquatch and added as metadata.
Sasquatch can use that timestamp to index the metrics database to allow correlations with the EFD data.
In addition, the timestamp of the metric calculation is going to be recorded as another field in InfluxDB and can also be used as the time axis for visualizations in Chronograf dashboards if needed. This is already possible using the `Flux query language`_ in InfluxDB 1.x.
InfluxDB 2.x supports new `visualization types`_ like scatter plots, histograms, and heatmaps that can also be useful.

.. _visualization types: https://docs.influxdata.com/influxdb/latest/visualize-data/visualization-types/
.. _Flux query language: https://docs.influxdata.com/flux/latest/get-started/

Tract/patch metrics are currently supported, although the time associated with such a metric must be solely the time of calculation.
There is also an additional time that can be derived from the software version itself.

The advantages of using Sasquatch are that InfluxDB and Chronograph are already optimized to support very large time-series datasets and also support query systems that allow metrics to be retrieved quickly into a Jupyter notebook using the EFD client software.

The Sasquatch developers are committed to adding additional metadata to support metrics calculated from pipelines tasks by data release production or the prompt processing or summit systems.

Tightly-couple Butler/Metric System
===================================

The possibility of storing metrics in a place that can be used by the graph building system, which really means dataset query system, has been discussed.
This would require the creation of a new datastore backend that uses the opaque table system to create tables in the main registry database.
These tables would then have to be published to the registry to allow registry to make use of them in queries.
Given that campaign tooling :cite:`DMTN-220` is taking a more general look at constraining inputs based on external databases, for example querying the EFD to determine the set of relevant exposures, we are not considering such tight integration into butler at this time.


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
