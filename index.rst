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
The different use cases for processing data are described in :cite:`DMTN-111` and in this document we focus on data processing directly driven by data acquisition.

Data processing must be triggered in some way with enough information to be able to work out which data to process and which analysis pipeline to use.
The unique identifier for every observation is the "Observation Identifier" (aka "obsid" aka "image name") that is guaranteed by the observatory systems to be unique for every observation regardless of telescope (main telescope vs AuxTel), instrument, or which controller was involved to take the data.
This obsid is issued by the camera control system whenever data are acquired.
Additionally, there is a "grouping" concept that can be used by scripts to indicate that observations taken with the same group identifier are related.\ [*]_
The group ID is how standard visits will be observed with both snaps being part of the same group.
The group ID will be provided to each script by the ScriptQueue and will be uniquely defined for that execution of the script.
The system will guarantee that no observing script will ever be given the same group ID.

Since DM software always mediates data access via a Data Butler this leads to a requirement that raw data must always be retrievable from a butler with a Data ID solely containing the obsid.
Additionally the detector can be specified in order to process a subset of the acquired data, and if a group ID is given instead of an obsid, multiple observations can be processed together.

All data products generated using this system are treated as transient data products that will not be archived at the Data Facility and will not be required to include permanent provenance tracking.
The purpose is to provide early feedback to observing in the form of data quality assessments.

.. [*] In :cite:`LTS-836` the group ID is called a "transaction ID" since it is associated with a single queue transaction.

.. note::

  No LSST rapid analysis pipelines are required to co-add data incrementally.
  Pipelines should not co-add data as it arrives and should always defer co-addition until the final observation in the group has been taken.

Controlling Data Processing from Observing Scripts
==================================================

In the initial design of the OCS Controlled Pipeline System (OCPS) [*]_ the pipeline processing will be controlled directly by commands sent from the observing script to the OCPS CSC.
The commands will indicate which data to process and the name of the pipeline to execute but they will not include algorithmic code.
The data will be specified using a "dataId" to match standard Data Butler usage.
Once the pipeline completes an event will be published.

.. [*] In :cite:`LDM-148` this is called the "OCS-Controlled Batch Service."

The sequence of events for a typical observing script would then be:

* Send a command to the camera to take an image.
  This command should include the relevant groupId string as supplied from the ScriptQueue, and the observing type (such as science, bias, or flat).
* Wait for the event indicating that the exposure has been written to the OODS (see :cite:`DMTN-111`).
  This event will include the relevant obsid.
* Send a command to the OCPS requesting that this obsid be processed.
  This command should include the name of the pipeline to execute.
* Acquire more data and send processing requests for each one.
* When all data are acquired and single-frame processing has completed, send a command to the OCPS requesting that a co-add pipeline be executed with all data using the shared groupId.
* An observing script can finish as soon as it has sent the final co-add command to the OCPS.

In some cases a complex script might require multiple groups to be defined during execution.
To deal with this case Python observing scripts shall be given an API to generate a new unique group ID from the value supplied to the script from the ScriptQueue.

.. warning::

   Observing scripts for standard visits and alternate standard visits should never use a group ID that differs from that supplied by the ScriptQueue.
   This is because the prompt processing system uses the group ID to collate data and expects a specific value to appear that matches the value published by the ScriptQueue.

As pipelines execute, they will be able to calculate data quality assessment metrics and other results of interest such as centroids.
These can be published as telemetry to allow them to be displayed to the observer.
This telemetry can be received by the observing script that issued the data processing request, allowing the script to fine tune its observing strategy (such as adjusting the CBP pointing).
These metrics should be associated with the history of the exposure in the same way as observer comments are associated with specific observations.

.. note::

   Notebooks can also block waiting for the processed data events to arrive before doing bespoke commissioning visualization or further detailed analysis that is decoupled from observing.

The OCPS CSC
------------

The OCPS CSC will support one command, let's say ``processData``, used by observing scripts.
The command will need to take at least a dataId and a pipeline name, and can possibly also take configuration overrides.
The dataId looks like a ``dict`` and would be expected to include either an obsid or a group ID and possibly a detector number.
The CSC will publish an event when a processing request has been completed.
It will also publish any telemetry from the pipelines themselves.

The OCPS itself acts as a small interface between the OCS and a pipeline execution environment.
The Observing Script does not have to know how pipelines are executed, all it needs to know is that named pipelines can be executed and will report when they have completed.

In the interim before Gen 3 is fully deployed, the OCPS could be something that launches a command line task.
It could also be a full Gen 3 pipeline executed by ``pipetask``.
It is though important that the OCPS knows when processing completes and also knows whether processing succeeded.
There also needs to be a mechanism for funneling metrics from the pipelines to the OCPS so that they can be issued as telemetry.

Open Questions
--------------

There are some details that need to be considered:

* If there is a delay in ingesting the previous observation into the OODS, how does the observing script know that the event that was received is for the correct observation?
  Should it listen for the camera event indicating that the data were taken and read the obsid from that event?
* If the OCPS is still handling the previous processing request when the new command arrives what should happen?
  Should it return immediately with a busy status?
  Should it queue the request?
  Should it block until the prior processing has completed but timeout if it is taking too long?
* Should the OCPS check that the data really are in the OODS before accepting the command?
  There is nothing in the system to prevent an observing script from sending the process command before ingestion.
* Should the observing script wait for the event from the OCPS indicating that the data were processed to completion before taking more data?
  Generally it should not wait unless it is interested in the telemetry.
* If data fail processing for some reason should the observing script be expected to know about it?
  Should the completion event indicate whether the pipeline completed without error?
* If data are processed but some QA metric fails, should it be included in the co-add?
  Failing QA will result in the output data being stored in the Butler repository and would likely result in good pipeline exit status.
* Should metrics be published to the SQuaSH system in addition to being published as telemetry?
  Given the transient nature of the summit data products and the presence of the metrics in the EFD via telemetry, it may not be necessary to also store them in SQuaSH.
* Does SAL require that all metrics that could possibly be reported from a pipeline be declared in advance?
  Or can metrics be issued as a JSON document and allow any content?
  Does LOVE work in such a dynamic environment?

Camera Diagnostic Cluster
=========================

The diagnostic cluster is currently designed to be used by the camera system for their own analysis.
The command issued by the CCS to trigger processing could be anything and could include an explicit ingest of the data files into a diagnostic-cluster-specific butler followed by calls to DM Pipeline code.
If the analysis being performed is substantially the same as that being performed by the OCPS then there is scope for the systems to be combined into a single system.

.. note::

   If the diagnostic cluster is used as the compute environment for OCPS then it will be necessary for the single-frame processing pipelines that are executed to always generate the JPEGs required for the summit visualization system.

Conclusions
===========

A system is proposed to allow observing scripts to initiate data processing of data taken by that observing script.
It uses a well-defined interface to make processing requests and can wait for completion events or continue to take additional data.

Appendix A: Future Options
==========================

In the above design, the observing script makes individual requests to the processing system.
An alternative approach can also be considered whereby the OCPS listens directly to the OODS and automatically processes the data without being commanded to do so by the observing script.
The OCPS would issue an event when data are sent for processing and an event when processing has completed.
For standard survey observing this approach is more than adequate and simplifies observing scripts.
We may consider using this scheme during survey operations to allow observing scripting to be more decoupled from data processing, whilst allowing more direct control in commissioning observing.

For this system to work whilst allowing a specific pipeline to be specified (rather than working it out from the observing mode or survey name) the observing script would need to ensure that the pipeline name is stored in the file's FITS header (which can be part of the camera take images command).

A metadata driven processing system does become harder in the scenario where multiple observations must be combined.
The observing script needs some way to indicate that the set of data from that script execution is now complete and must be processed together.
One way of doing this is to include a "group has ended" boolean in the file header.
There is some subtlety over which pipeline should run for co-addition and whether it's a single pipeline that includes the single frame processing (in which case that pipeline should pause whilst new data arrive) or a distinct pipeline that takes the inputs from single frame processing and is related to the pipeline name specified in the header using some agreed upon convention.

If data arrives whilst previous data is being processed, the behavior on how to act could be configurable.
It's reasonable to ignore that data and only process data that arrive whilst the processing system is free.
If there are sufficient compute resources the data could be sent to the pipelines but load management would then have to be implemented to ensure that the system doesn't continue to back up even further.
Some data might be marked as always having to be processed and other data could be allowed to be skipped.

If the ScriptQueue is told to stop the observing script early such that the "group complete" flag will never turn up, the ScriptQueue can send a command to the OCPS to trigger the co-addition pipeline.
OCPS could also notice a change of group ID and be configured complete any processing.
The general approach though is that the observing script does not know anything about the OCPS but can control which observations are co-added and which pipeline to use by ensuring that the file headers include this information.
This metadata-mediated processing approach is similar to that used by ORAC-DR :cite:`2015A&C.....9...40J`.


.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
