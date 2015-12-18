.. _extracting_rows_from_the_pages:

==============================
Extracting Rows From The Pages
==============================

The constraints_parser tool works by scanning the data file one byte at a time, and checking every offset with ``check_for_a_record()`` to see if it looks like the beginning of a row. This checks the row's origin, and the field sizes and so forth. These definitions are set in the :file:`include/table_defs.h` file.

If the data at the current position appears to be a valid row, then the tool applies ``check_constraints()``, which is a kind of ``WHERE`` clause, to the row and determines whether to accept it. This is specified in each column's limits section in :file:`include/table_defs.h`. We will look more at this later.

Concatenating Pages Into a Single File
======================================

If your server doesn't have :variable:`innodb_file_per_table` enabled, then before you run the tool, you need to determine which index ID contains our data, as we've seen previously. In our example, we needed index ID ``0 286``, and the pages containing our data are in the ``pages-1246363747/0-286/`` directory.

.. code-block:: bash

  total 120
  -rw-r--r-- 1 root root 16384 Jun 30 05:09 1254-00001254.page
  -rw-r--r-- 1 root root 16384 Jun 30 05:09 1255-00001255.page
  -rw-r--r-- 1 root root 16384 Jun 30 05:09 1256-00001256.page
  -rw-r--r-- 1 root root 16384 Jun 30 05:09 1257-00001257.page
  -rw-r--r-- 1 root root 16384 Jun 30 05:09 50-00000050.page
  -rw-r--r-- 1 root root 16384 Jun 30 05:09 74-00000050.page

It is possible to run the tool against individual pages, and in fact that is helpful initially to ensure that your table definition is tuned optimally. When you are satisfied with the table definition, it is more useful to run it against all of the pages concatenated in physical order. You can create such a file with the cat command:

.. code-block:: bash

  $ find pages-1246363747/0-286/ -type f -name '*.page' | sort -n \
     | xargs cat > pages-1246363747/0-286/customer_pages_concatenated

The resulting file, :file:`pages-1246363747/0-286/customer_pages_concatenated`, is what we will use as input to the ``constraints_parser`` tool. If you use :variable:`innodb_file_per_table`, you can use the table's :file:`.ibd` file instead.

Running the ``constraints_parser`` Tool
=======================================

The core step in recovering data is to run the ``constraints_parser`` tool against the data and extract the rows. As with the ``page_parser`` tool, you must specify the page format (``COMPACT`` or ``REDUNDANT``), and the ``-f`` option specifies the input file. The tool prints tab-separated, quote-delimited values to standard output.

Returning to our running example, we can execute the tool as follows (for this initial run, we are looking only at a single page, not the file containing all the pages concatenated together):

.. code-block:: bash

  $ ./constraints_parser -5 -f pages-1246363747/0-286/50-00000050.page

The output is lengthy, so we abbreviate here for demonstration purposes. Some of the output seems correct; some is obviously garbage; and some requires knowledge of the data to assess. We will see in the next section how to tune the table definition so that the tool outputs as much valid data as possible and filters out the garbage.

The output columns are the table name, and the columns in the table.

.. code-block:: shell

  customer        0       120     ""      ""      ""      32770   0       "0000-00-00 00:12:80"   0
  customer        0       0       ""      ""      ""      0       0       "9120-22-48 29:44:00"   2
  customer        61953   0       ""      ""      ""      2816    0       "7952-32-67 11:43:49"   0
  customer        0       0       ""      ""      ""      0       0       "0000-00-00 00:00:00"   0
  ... snip ...
  customer        0       0       ""      ""      ""      0       0       "0000-00-00 00:00:00"   16777728
  customer        28262   114     ""      ""      NULL    25965   117     "4603-91-96 76:21:28"   5111809
  customer        0       82      ""      ""      ""      22867   77      "2775-94-58 03:19:18"   1397573972
  customer        2       1       "PATRICIA"      "JOHNSON"       "PATRICIA.JOHNSON@sakilacustomer.org"   6       1       "2006-02-14 22:04:36"   1140008240
  customer        3       1       "LINDA" "WILLIAMS"      "LINDA.WILLIAMS@sakilacustomer.org"     7       1       "2006-02-14 22:04:36"   1140008240
  customer        4       2       "BARBARA"       "JONES" "BARBARA.JONES@sakilacustomer.org"      8       1       "2006-02-14 22:04:36"   1140008240
  customer        5       1       "ELIZABETH"     "BROWN" "ELIZABETH.BROWN@sakilacustomer.org"    9       1       "2006-02-14 22:04:36"   1140008240
  customer        6       2       "JENNIFER"      "DAVIS" "JENNIFER.DAVIS@sakilacustomer.org"     10      1       "2006-02-14 22:04:36"   1140008240
  customer        7       1       "MARIA" "MILLER"        "MARIA.MILLER@sakilacustomer.org"       11      1       "2006-02-14 22:04:36"   1140008240
  customer        8       2       "SUSAN" "WILSON"        "SUSAN.WILSON@sakilacustomer.org"       12      1       "2006-02-14 22:04:36"   1140008240
  customer        9       2       "MARGARET"      "MOORE" "MARGARET.MOORE@sakilacustomer.org"     13      1       "2006-02-14 22:04:36"   1140008240
  ... snip ...
  customer        0       0       ""      ""      ""      0       0       "0000-00-00 00:00:00"   0
  customer        0       0       ""      ""      ""      0       0       "7679-35-98 86:44:53" 
