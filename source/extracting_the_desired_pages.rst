.. _extracting_the_desired_pages:

============================
Extracting the Desired Pages
============================

The tools work by trying to find valid rows in pages. |InnoDB|'s pages are 16k in size by default, and each page belongs to a specific index in a specific table. It can save a lot of time and work to determine which pages belong to which tables and indexes, and only try to extract rows from the pages that interest you and whose row format should match the index you're trying to recover. It will do you no good to try to extract an index's data from pages that don't belong to that index. You can get a lot of confusing false positives, and the tools can be much slower due to checking and re-checking pages for row matches for every possible table.

To accomplish the task of separating the pages out, the toolkit contains a tool called ``page_parser``. This tool reads a file and copies each page into a separate file in a subdirectory according to the index ID in the page header.

If your server is configured with :variable:`innodb_file_per_table` set to ``1``, then the work is probably done for you already. The pages of interest are all in the :file:`.ibd` file, and you do not need to split it out in most cases. However, even that :file:`.ibd` file potentially contains multiple indexes, so it can still be beneficial to split the pages out separately. If the server is not configured with :variable:`innodb_file_per_table` enabled, then the data is stored in the global tablespace (usually in a file called :file:`ibdata1`), and it is usually a good idea to split the file up into its component pages.

Splitting The Pages Apart
=========================

To separate the pages, execute ``page_parser``. The following snippet shows a sample of the tool's usage. If the data is in |InnoDB|'s old ``REDUNDANT`` format (pre-MySQL-5.0), then you need to give the -4 argument instead of -5, which is for |MySQL| 5.0's ``COMPACT`` format.

.. code-block:: bash

  page_parser -5 -f /path/to/ibdata1
  [snip]
  Read page #494.. saving it to pages-1246363747/0-97/494-00000494.page
  Read page #495.. saving it to pages-1246363747/0-97/495-00000495.page
  Read page #496.. saving it to pages-1246363747/0-97/496-00000496.page
  Read page #497.. saving it to pages-1246363747/0-97/497-00000497.page
  Read page #498.. saving it to pages-1246363747/0-97/498-00000498.page
  Read page #499.. saving it to pages-1246363747/0-96/499-00000499.page
  [snip]

The tool prints one line per page to standard output, and occasionally it prints a progress indicator to standard error.

The ``page_parser`` tool creates a directory called ``pages-<TIMESTAMP>``, with the current Unix timestamp as the last part of the name. Within this directory, it creates one directory for every index ID it finds in the data file's pages. These subdirectories are named after the page's index ID, in high-low format (InnoDB's index IDs are 64-bit integers composed of a high 32 bits and a low 32 bits). Within the index subdirectory, the pages are numbered by an incrementing integer, and the page's internal ID number. For example:

.. code-block:: bash

  pages-1301110359/0-52/
  pages-1301110359/0-52/216-00000216.page
  pages-1301110359/0-52/70-00000216.page

The file :file:`70-00000216.page` was read before the file :file:`216-00000216.page`; this physical ordering can be important to preserve.

Determining the Desired Index ID
================================

Now that we have split the pages into separate subdirectories, we need to determine which index holds the data we are interested in recovering. In most cases, we want to recover from the table's PRIMARY index, which contains the full rows. There are several ways to accomplish this.

If the database server is still running, and the table has not been dropped, you can start the InnoDB Tablespace Monitor and let it write out all of the tables and indexes, and their index IDs, to the server's error log file. You can do that by creating a magically named table:

.. code-block:: mysql

  mysql> CREATE TABLE innodb_table_monitor (id int) ENGINE=InnoDB;

If the table already exists, ``DROP`` it, then ``CREATE`` it again. Wait for up to a minute, then look at the |MySQL| error log for output. Once the output comes, you can drop the special table to stop the tablespace monitor from printing more output. A summary of the output follows:

.. code-block:: bash

  TABLE: name sakila/customer, id 0 142, columns 13, indexes 4, appr.rows 0
    COLUMNS: customer_id: DATA_INT len 2 prec 0; store_id: DATA_INT len 1 prec 0; first_name: type 12 len 135 prec 0; last_name: type 12 len 135 prec 0; email:
   type 12 len 150 prec 0; address_id: DATA_INT len 2 prec 0; active: DATA_INT len 1 prec 0; create_date: DATA_INT len 8 prec 0; last_update: DATA_INT len 4 pr
   ec 0; DB_ROW_ID: DATA_SYS prtype 256 len 6 prec 0; DB_TRX_ID: DATA_SYS prtype 257 len 6 prec 0; DB_ROLL_PTR: DATA_SYS prtype 258 len 7 prec 0; 
   INDEX: name PRIMARY, id 0 286, fields 1/11, type 3
     root page 50, appr.key vals 0, leaf pages 1, size pages 1
     FIELDS:  customer_id DB_TRX_ID DB_ROLL_PTR store_id first_name last_name email address_id active create_date last_update
   INDEX: name idx_fk_store_id, id 0 287, fields 1/2, type 0
     root page 56, appr.key vals 0, leaf pages 1, size pages 1
     FIELDS:  store_id customer_id
   INDEX: name idx_fk_address_id, id 0 288, fields 1/2, type 0
     root page 63, appr.key vals 0, leaf pages 1, size pages 1
     FIELDS:  address_id customer_id
   INDEX: name idx_last_name, id 0 289, fields 1/2, type 0
     root page 1493, appr.key vals 0, leaf pages 1, size pages 1
     FIELDS:  last_name customer_id

For our recovery example, we're looking for the ``sakila/customer`` table's ``PRIMARY KEY`` details: ``INDEX: name PRIMARY, id 0 286, fields 1/11, type 3``. The index ID is ``0 286``. This tells us that the pages we need are in the ``0-286`` subdirectory.
