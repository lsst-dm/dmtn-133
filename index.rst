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

.. _introduction:

Introduction
============

At the summit it is sometimes desirable for instrument data to be analyzed rapidly after data acquisition in order to get quick feedback of data quality.
The different use cases for processing data are described in :cite:`DMTN-111` and in this document we focus on data processing directly related to data acquisition.

Data processing must be triggered in some way with enough information to be able to work out which data to process and which analysis pipeline to use.
The unique identifier for every observation is the "Observation Identifier" (aka "obsid" aka "image name") that is guaranteed by the observatory systems to be unique for every observation regardless of telescope (main telescope vs AuxTel), instrument, or which controller was involved to take the data.
This obsid is issued by the camera control system whenever data are acquired.
Additionally, there is a "grouping" concept that can be used by scripts to indicate that observations taken with the same group identifier are related [*]_.
The group ID is how standard visits will be observed with both snaps being part of the same group.
The group ID will be provided to each script by the ScriptQueue and will be unique for that execution of the script.

Since DM software always mediates data access via a Data Butler this leads to a requirement that raw data must always be retrievable from a butler with a Data ID solely containing the obsid, and optionally, the detector name or group name.
:cite:`DMTN-111` also states that some event must be issued whenever new data have been ingested and that that event must contain the obsid.

.. [*] In :cite:`LTS-836` the group ID is called a "transaction ID" since it is associated with a single queue transaction.

Processing Data as it Arrives
=============================

All triggered processing should be done through the "Gen3" middleware pipeline system consisting of a pipeline of``PipelineTask``.
Each trigger will need to specify at minimum the name of the Pipeline to execute and the Data ID that should be processed.
Additionally it may be necessary to provide an explicit config file by name.
Using standard change-controlled pipeline configurations is preferable to scripts trying to send override configuration on-the-fly since it provides a self-contained way to reproduce the processing.

For a Pipeline to be executed the Data ID needs to be converted to a quantum graph suitable for passing to the activator.
The Data ID would usually be the obsid for the data just acquired.
Furthermore the group ID could be used to obtain all data that have previously been taken related to this observation.

.. note::

  No LSST rapid analysis pipelines are required to co-add data incrementally.
  Pipelines should not co-add data as it arrives and should always defer co-addition until the final observation in the group has been taken.

Single frame processing is the simplest mode whereby the products from the processing are not to be combined with subsequent observations.
The more complex scenario relates to processing that requires co-addition where the co-addition can only occur at the end of data acquisition.

One proposal for this is to have a chained pipeline approach.
As data arrives they are processed with the nominated single frame processing variant.
On completion they register themselves as inputs to a combination pipeline.
This pipeline does not run automatically but waits to be told that all data have arrived.
If an observation fails to reduce properly, or fails to pass some metrics threshold it will not register with the combination pipeline.

When all expected data have arrived and been processed on their own, the final combination pipeline will be unblocked and will run to completion.
Events will be issued on the completion of a single frame processing and on the completion of the combination pipeline.
Metrics will also be published where appropriate during processing and made available as SAL events.

The issues that have to be solved are:

1. How does the pipeline know to start processing a Data ID?
2. Which pipeline should be run for the given Data ID?
3. If the pipeline supports chaining, is that known to the system passing in the data?
4. Who decides which observations should be combined and how?
5. How does the system know the final observation arrived such that the combination pipeline can run?

This document will describe two scenarios for answering the above questions.
The first describes a system entirely controlled by the observing script that is running the observation.
The second describes a system that is decoupled from the observing system and is entirely driven by data arriving in the system.

To simplify the remainder of this document the name given to the system that forwards processing requests to pipelines will be the "OCS Controlled Pipeline System (OCPS)."

Observing Script Control
========================

In this scenario the observing script or notebook [*]_ has full control of all processing and the observing sequence and data processing are tightly coupled.
The control flow could look something like this:

* Tell OCPS that a new co-addition pipeline with the defined name is to be readied and that any previous pending data should be flushed.
* Loop over however many images are to be taken
   * Send the "take images" command to the camera, including the group ID issued by the ScriptQueue.
   * Asynchronously wait for the "data available" message to come from the Archiver or CCS containing the obsid.
      - Send command to OCPS requesting that data with that obsid be processed.

* Once loop completes and all obsids have been sent for processing, send command to OCPS telling it to run the co-addition pipeline.

If the script is told to stop early the script can send the command to run the co-addition pipeline as part of its clean up phase.

In this system the group ID is not used to determine whether an observation should be co-added and its role is solely to indicate that the data were taken by that particular execution of the script.
The scripting approach does not store any additional metadata with the data and therefore subsequent offline processing can not reproduce exactly the grouping specified by the observing script without examining the provenance in the products themselves.

.. [*] Hereafter the term "script" will be used to include Jupyter notebooks unless explicitly stated otherwise.

Metadata Driven Processing
==========================

In this scenario the data processing is completely decoupled from the observing script and decisions on the processing within the OCPS are entirely mediated by metadata associated with the observation itself.
The control flow in the observing script could look something like this:

* Script sends multiple "take images" commands in loop.
   * This command includes specification of a groupid and pipeline name.
   * If this is for a co-addition pipeline, for the final observation in group also include a flag indicating that data taking in this group is complete.

and the OCPS will asynchronously process the data that arrives:

* The Archiver/CCS issues a "data available" event with the relevant obsid once the data are available in a butler repository.
* OCSP triggers on this "data available" event.
* OCPS retrieves relevant metadata from the butler using this obsid.
* Determine the pipeline name from the metadata.
* Check whether this is from a previously known groupid or a new groupid.
* If this is a new groupid ensure that any previous co-addition pipelines have been flushed.
* Create quantum graph for this pipeline and send for execution.
* If the "group complete" flag is set in the metadata, trigger the co-addition pipeline.

The script has the option of changing the groupid in order to control how data are grouped in the OCPS.
A standard mechanism will be provided to scripts for generating a new unique groupid from an existing groupid.
A script should not modify the supplied for standard visits or alternate standard visits since the Alert Production system uses the groupid issued by the ScriptQueue as a rendezvous mechanism.

If data arrives whilst previous data is being processed, the behavior on how to act could be configurable.
It's reasonable to ignore that data and only process data that arrive whilst the processing system is free.
If there are sufficient compute resources the data could be sent to the pipelines but load management would then have to be implemented to ensure that the system doesn't continue to back up even further.
Some data might be marked as always having to be processed and other data could be allowed to be skipped.

If the ScriptQueue is told to stop the observing script early such that the "group complete" flag will never turn up, the ScriptQueue can send a command to the OCPS to trigger the co-addition pipeline.
The general approach though is that the observing script does not know anything about the OCPS but can control which observations are co-added and which pipeline to use.
This metadata-mediated processing approach is similar to that used by ORAC-DR :cite:`2015A&C.....9...40J`.

Using Processed Data
====================

Once data have been processed by the OCPS an event should be issued when each dataset type becomes available.
The event could trigger the observatory visualization system allowing specific dataset types and detectors to be selected for immediate display as part of the summit quality control process.
The observatory visualization system configuration can therefore be loosely coupled to the OCPS and how to display a particular dataset type is entirely configuration driven within the visualization system.

Notebooks can also block waiting for the processed data before doing bespoke commissioning visualization or further detailed analysis that is decoupled from observing.

Metrics from processing should be published to the EFD and/or to SQuaSH :cite:`SQR-009` and should be available to scripts to allow them to make decisions on how best to take more data.

Camera Diagnostic Cluster
=========================

The diagnostic cluster is currently designed to be used by the camera system for their own analysis.
The command issued by the CCS to trigger processing could be anything and could include an explicit ingest of the data files into a diagnostic-cluster-specific butler followed by calls to DM Pipeline code.
If the analysis being performed is substantially the same as that being performed by the OCPS then there is scope for the systems to be combined into a single system.

Conclusions
===========

Two approaches for triggering pipeline processing of data have been presented.
The scripting approach looks extremely flexible but results in significantly more complex observing scripts.
Metadata driven processing isolates the complexity in a single place and results in simpler observing scripts whilst still supporting all the use cases.
It may seem that the latter approach is more constraining but it provides a small, well-defined interface between data acquisition and data processing whilst retaining sufficient flexibility to support rapid triggered data processing.



.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
