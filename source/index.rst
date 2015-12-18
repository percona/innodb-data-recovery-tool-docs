.. InnoDB Data Recovery Tool documentation master file, created by
   sphinx-quickstart on Fri Dec 19 15:45:46 2015.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

.. _dochome:

=========================================
InnoDB Data Recovery Tool - Documentation
=========================================

Percona's Data Recovery tool for |InnoDB| is a set of tools that can help recover data from lost or corrupted |MySQL| tables. It works by extracting the rows from the raw files.

There are no guarantees, especially in case of corruption that affects the row storage areas of the files (as opposed to the file headers and other meta-data). It is a tedious process that requires an understanding of InnoDB's internal data storage format, C programming, and quite a bit of intuition and experience. If you are having trouble using the tools, |Percona| provides |InnoDB| `data recovery services <https://www.percona.com/services/consulting>`_ with a team of skilled and experienced technicians, and is available 24 hours a day to assist you.

Detailed documentation is in the tool's manual.

Full Table of Contents
=======================

.. toctree::
   :maxdepth: 1
   
   intro
   prerequisites
   example_data_loss_scenario
   building_the_tools
   extracting_the_desired_pages
   generating_a_table_definition
   extracting_rows_from_the_pages
   fine_tuning_the_table_definition
   importing_data_into_the_database
   advanced_recovery_techniques
   external_resources
   copyright
   trademark-policy


