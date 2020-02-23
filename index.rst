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
Additionally, there is a "grouping" concept that can be used by scripts to indicate that observations taken with the same group identifier are related.\ [#f1]_
The group ID is how standard visits will be observed with both snaps being part of the same group.
The group ID will be provided to each script by the ScriptQueue and will be uniquely defined for that execution of the script.
The system will guarantee that no observing script will ever be given the same group ID.

Since DM software always mediates data access via a Data Butler this leads to a requirement that raw data must always be retrievable from a butler with a Data ID solely containing the obsid.
Additionally the detector can be specified in order to process a subset of the acquired data, and if a group ID is given instead of an obsid, multiple observations can be processed together.

All data products generated using this system are treated as transient data products that will not be archived at the Data Facility and will not be required to include permanent provenance tracking.
The purpose is to provide early feedback to observing in the form of data quality assessments.

.. note::

  No LSST rapid analysis pipelines are required to combine data incrementally.
  Pipelines should not combine data as it arrives and should always defer group processing until the final observation in the group has been taken.

Controlling Data Processing from Observing Scripts
==================================================

In the initial design of the OCS Controlled Pipeline System (OCPS) [#f2]_ the pipeline processing will be controlled directly by commands sent from the observing script to the OCPS CSC.
The commands will indicate which data to process and the name of the pipeline to execute but they will not include algorithmic code.
OCPS should be configurable to allow development pipelines and development science pipelines packages to be used.
The data will be specified using a "dataId" to match standard Data Butler usage.
Once the pipeline completes an event will be published.

The sequence of events for a typical observing script would then be:

* Send a command to the camera to take an image.
  This command should include the relevant groupId string as supplied from the ScriptQueue, and the observing type (such as science, bias, or flat).
* Wait for the event indicating that the exposure has been written to the OODS (see :cite:`DMTN-111`).
  This event will include the relevant obsid.
* Send a command to the OCPS requesting that this obsid be processed.
  This command should include the name of the pipeline to execute, possibly the name of a configuration file to use and maybe explicit overrides of default configurations.
* Acquire more data and send processing requests for each one.
* When all data are acquired and single-frame processing has completed, send a command to the OCPS requesting that a pipeline be executed combining all data using the shared groupId.\ [#f3]_
* An observing script can finish as soon as it has sent the final combination command to the OCPS.

In some cases a complex script might require multiple groups to be defined during execution.
To deal with this case Python observing scripts shall be given an API to generate a new unique group ID from the value supplied to the script from the ScriptQueue.

.. warning::

   Observing scripts for standard visits and alternate standard visits should never use a group ID that differs from that supplied by the ScriptQueue.
   This is because the prompt processing system uses the group ID to collate data and expects a specific value to appear that matches the value published by the ScriptQueue.

As pipelines execute, they will be able to calculate data quality assessment metrics and other results of interest such as centroids.
These can be published as telemetry to allow them to be displayed to the observer.
This telemetry can be received by the observing script that issued the data processing request, allowing the script to fine tune its observing strategy (such as adjusting the CBP pointing).
These metrics should be associated with the history of the exposure in the same way as observer comments are associated with specific observations.

Image display of resulting datasets should not be under the control of the observing script.
Scripts should not have to know whether an observer is interested in seeing an image or not.
Ideally an asynchronous display mediator task should exist that can be configured to be interested in specific dataset types.
It can either be contacted directly by the running pipelines ("I've just created dataset X") and decide whether it's interested, or else it can monitor a butler repository waiting for datasets to appear.

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

Design Decisions
----------------

* Observing scripts should listen to the ``startIntegration`` event from the camera to ensure that they are listening for the correct data being available in the OODS (OODS will not know about the groupId but will know about obsid).
* It should be possible for the OCPS to receive a processing command whilst a previous processing job is running.
* The OCPS can run in a synchronous or asynchronous mode
  In synchronous mode only a single job can be handled at a time with other requests rejected and the command completing when the processing completes.
  In asynchronous mode multiple processing requests can be queued and separate messages are issued as they complete.
* If a command is sent to the OCPS before the relevant data have been ingested, pipelines should not be held back but failure from the pipelines should be reported.
* When an observing script sends its final command to the OCPS it should not wait for processing to complete unless the script requires that the outcome of the processing affects further observing.
* Completion events from OCPS should indicate whether processing was successful or not.
* QA results are not reported by the OCPS but must be monitored independently.
  It is likely that failing QA would still result in a good completion status from the pipeline.
* Metrics should be published through the EFD in the same form as those published by Prompt Processing.
  Additionally some mechanism maybe included for integrating metrics into the SQuaSH system. :cite:`SQR-009`

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

.. rubric:: Footnotes

.. [#f1] In :cite:`LTS-836` the group ID is called a "transaction ID" since it is associated with a single queue transaction.
.. [#f2] In :cite:`LDM-148` this is called the "OCS-Controlled Batch Service."
.. [#f3] It's an open question whether the pipeline should be preconfigured to expect a fixed number of exposures to arrive and then automatically run the "gather" step, or if the observing script should trigger the "gather" step.
       In the initial design it is simpler to be explicit.


.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
