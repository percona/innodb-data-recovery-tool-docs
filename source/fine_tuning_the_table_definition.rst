.. _fine_tuning_the_table_definition:

================================
Fine Tuning The Table Definition
================================

The table definition in the *C* struct contained in :file:`include/table_def.h` can be tuned to not only specify the correct structure and data types in the table, but also to specify constraints on the column values. This can help the tool reject invalid data and give better results.

Recognizing invalid data is not simply a matter of filtering out bad rows and cutting down on noise. If the tool believes that a row begins at a certain spot, it will accept a range of data as a row, and will not begin searching for the start of a new row until after the end of the range – it will advance the current position by the size of the row it found. Therefore, if it thinks it finds a row in a place where there is none, and a real row's origin lies within the range, the tool will not find the real row. For that reason, tuning the table definition is an important part of finding valid data, not simply filtering out false positives. **Finding a bad row usually means failing to find a good row.**

It can be extremely helpful to run ``constraints_parser`` in verbose mode, with the ``-V`` command-line option, to inspect the process of checking data to see if it appears to be a row, and why or why not. The output is very verbose, so you will generally want to do this only on a single page, not on a large file.

In our ongoing example, the tool printed out some rows whose ``customer_id`` is ``0``, some with empty first names, and so on. We know that this is not correct. All customers have positive integer ``customer_id`` columns, and all of them have non-empty names. To instruct the tool to reject rows unless they meet these criteria, we can change the limits clause in the table definitions header. First, we specify that ``customer_id`` must be greater than zero:

.. code-block:: c

  { /* smallint(5) unsigned */
        name: "customer_id",
        type: FT_UINT,
        fixed_length: 2,
        has_limits: TRUE,
        limits: {
                can_be_null: FALSE,
  -             uint_min_val: 0,
  +             uint_min_val: 1,
                uint_max_val: 65535
        },
        can_be_null: FALSE
  },

Next, we specify that first_name must be at least one character long. This might still be too relaxed; is there really a customer whose first name is a single letter? The better you know your data, the more precisely you will be able to specify the constraints. In reality the minimum length in the column is 2, and the maximum is 11, even though the column is defined as ``VARCHAR(45)``. Setting the maximum length would be a good idea, if we knew it. For now, we will simply forbid empty first names:

.. code-block:: c

  { /* varchar(45) */
        name: "first_name",
        type: FT_CHAR,
        min_length: 0,
        max_length: 45,
        has_limits: TRUE,
        limits: {
                can_be_null: FALSE,
  -             char_min_len: 0,
  +             char_min_len: 1,
                char_max_len: 45,
                char_ascii_only: TRUE
        },
        can_be_null: FALSE
  }

After making this change, you should rebuild and run the ``constraints_parser`` tool again. This time the output looks much better:

.. code-block:: bash

  $ ./constraints_parser -5 -f pages-1246363747/0-286/50-00000050.page
  customer        1       1       "MARY"  "SMITH" "MARY.SMITH@sakilacustomer.org" 5       1       "2006-02-14 22:04:36"   1140008240
  customer        2       1       "PATRICIA"      "JOHNSON"       "PATRICIA.JOHNSON@sakilacustomer.org"   6       1       "2006-02-14 22:04:36"   1140008240
  customer        3       1       "LINDA" "WILLIAMS"      "LINDA.WILLIAMS@sakilacustomer.org"     7       1       "2006-02-14 22:04:36"   1140008240
  customer        4       2       "BARBARA"       "JONES" "BARBARA.JONES@sakilacustomer.org"      8       1       "2006-02-14 22:04:36"   1140008240
  customer        5       1       "ELIZABETH"     "BROWN" "ELIZABETH.BROWN@sakilacustomer.org"    9       1       "2006-02-14 22:04:36"   1140008240
  customer        6       2       "JENNIFER"      "DAVIS" "JENNIFER.DAVIS@sakilacustomer.org"     10      1       "2006-02-14 22:04:36"   1140008240
  customer        7       1       "MARIA" "MILLER"        "MARIA.MILLER@sakilacustomer.org"       11      1       "2006-02-14 22:04:36"   1140008240
  customer        8       2       "SUSAN" "WILSON"        "SUSAN.WILSON@sakilacustomer.org"       12      1       "2006-02-14 22:04:36"   1140008240
  customer        9       2       "MARGARET"      "MOORE" "MARGARET.MOORE@sakilacustomer.org"     13      1       "2006-02-14 22:04:36"   1140008240
  customer        10      1       "DOROTHY"       "TAYLOR"        "DOROTHY.TAYLOR@sakilacustomer.org"     14      1       "2006-02-14 22:04:36"   1140008240
  customer        11      2       "LISA"  "ANDERSON"      "LISA.ANDERSON@sakilacustomer.org"      15      1       "2006-02-14 22:04:36"   1140008240
  customer        12      1       "NANCY" "THOMAS"        "NANCY.THOMAS@sakilacustomer.org"       16      1       "2006-02-14 22:04:36"   1140008240
  ...
  customer        596     1       "ENRIQUE"       "FORSYTHE"      "ENRIQUE.FORSYTHE@sakilacustomer.org"   602     1       "2006-02-14 22:04:37"   1140008240
  customer        597     1       "FREDDIE"       "DUGGAN"        "FREDDIE.DUGGAN@sakilacustomer.org"     603     1       "2006-02-14 22:04:37"   1140008240
  customer        598     1       "WADE"  "DELVALLE"      "WADE.DELVALLE@sakilacustomer.org"      604     1       "2006-02-14 22:04:37"   1140008240
  customer        599     2       "AUSTIN"        "CINTRON"       "AUSTIN.CINTRON@sakilacustomer.org"     605     1       "2006-02-14 22:04:37"   1140008240

It appears that this change has enabled the tool to recover all of the rows correctly. You can now run the tool against all of the pages, and redirect the output to a file, which you can load into the database to restore the table.

.. code-block:: bash

  $ ./constraints_parser -5 -f pages-1246363747/0-286/customer_pages_concatenated \
     > /tmp/customer_data.tsv

Handling Duplicates and Old Row Versions
========================================

The file that results from the previous step has only 689 records, instead of the 1416 before specifying the constraints more accurately. However, this is still more than the actual number in the table, which is 599. What are these rows?

Sometimes |InnoDB| marks a row as deleted, or writes a new version of a row but keeps the old version and marks it as deleted, resulting in two versions of the row in the output. Depending on the type of data loss you are trying to recover, you might need to ignore those row versions – or perhaps they are the data you are looking for. These two cases can be handled as follows:

 * Because we sorted the pages in physical order to concatenate them, obsolete rows should come first and the newest row version should come last in the output.
 * Deleted rows can be output by running the tool with the -D option. You can then remove these from the final results.

Sometimes there are still garbage rows in the output, even with good constraints and after de-duplicating and removing old versions. The output of this tool usually requires careful validation.

Understanding Table Definitions
===============================

The tool supports a variety of data types, and filters (“limits”) on the data, but not everything is supported. Examine the header file :file:`include/tables_dict.h` to see what is supported. Briefly,

  1) The tool does not support ``SET``, ``BLOB``, or ``BIT`` data types. ``ENUM`` is supported.

  2) ``CHAR`` and ``TEXT`` are supported, as long as off-page storage is not used.

  3) For numeric types, you can specify min and max values.

  4) For character types, you can specify min and max length.

  5) For character types, you can additionally restrict the values to ASCII, digits, or match them against a regular expression.

  6) Date types can be validated as dates.

There are some special cases and things you need to know for specific data types.

Defining and Constraining Data Types
====================================

The following data types and constraints are supported:

All Data Types
--------------


 * Any field can be set nullable or non-nullable: ``can_be_null: TRUE|FALSE``
 * The ``has_limits`` member defines whether the limits are applied: ``has_limits: TRUE``

The DECIMAL Data Type
---------------------

DECIMAL types are stored as strings before MySQL 5.0.3, and in a packed format after that version. The :program:`create_defs.pl` tool generates a definition for |MySQL| 5.0.3 and later. To recover DECIMAL from tables in an earlier version of |MySQL|, use ``FT_CHAR`` field type. For a column defined as DECIMAL(M, D), use the following rules to choose a length in the table definition:

 * if D > 0, then M+2 bytes
 * if D = 0, then M+1 bytes
 * if M < D, then D+2 bytes

Here is an example definition for a DECIMAL(4,2) column:

.. code-block:: c

  { /* decimal(4,2) unsigned */
        name: "my_decimal_column",
        type: FT_CHAR,
        fixed_length: 6,
        has_limits: FALSE,
        limits: {
                can_be_null: TRUE,
                char_min_len: 0,
                char_max_len: 2,
                char_ascii_only: TRUE
        },
        can_be_null: TRUE
  },

The INT and INT UNSIGNED Data Type
----------------------------------

The INT type permits the following constraints: ``int_min_val`` and ``int_max_val``. Unsigned integers permit ``uint_min_val`` and ``uint_max_val``.

The TIMESTAMP Data Type
-----------------------

This data type is stored as ``UINT`` by |InnoDB|.

The DATETIME Data Type
----------------------

DATETIME types are stored as 8-byte integers internally. The tool recognizes them, and you can specify that they must be validated as dates to be accepted. If you set ``has_limits: TRUE``, then the tool will enforce that seconds and minutes are in the interval 0..59, hours range from 0..23, days range from 1..31, months range from 1..12, and years range from 1950..2050.

The CHAR, VARCHAR, and TEXT Data Types
The following constraints can be applied:

 * char_min_len
 * char_max_len
 * char_ascii_only
 * char_digits_only
 * char_regex

Here is an example to constrain values to only URLs containing ASCII codes, between 20 and 100 characters in length:

.. code-block:: c

  char_min_len: 20,
  char_max_len: 100,
  char_ascii_only: TRUE,
  char_regex: "^http://",
                        
Although the ``char_regex`` constraint could be used to mimic other types of constraints, it is much more CPU intensive, so it should only be used when a more efficient alternative is not available.

The BLOB Data Type
------------------

Due to the complexities of ``BLOB`` storage, the tool does not yet support ``BLOB`` columns. However, if the values are short enough, |InnoDB| actually stores them as strings, and even if the value is too long and is stored off-page in a separate segment, a prefix of the value is stored on-page as a string, and if the record fits in the page, the whole value might be stored on the page. (This is not the case with the Barracuda page format, however.)

To recover ``BLOB`` values, wholly or in part, you can sometimes simply treat it as ``FT_TEXT``. You can patch the :file:`create_defs.pl` file to make it treat ``BLOB`` as ``TEXT``:

.. code-block:: c

  sub FindFieldByName($$) {
        my $fields = shift;
        my $name = shift;
  @@ -333,7 +346,7 @@
        return { type => 'FT_TEXT', min_len => 0, max_len => 255 };
        }
  -   if ($type =~ /^TEXT$/i) {
  +   if ($type =~ /^(TEXT|BLOB)$/i) {
        return { type => 'FT_TEXT', min_len => 0, max_len => 65535 };
  }

The ENUM Data Type
------------------

Here is a sample definition for an ``ENUM``, which verifies that the enum's INT representation is less than or equal to 2. The ``enum_values`` limit is not really a limit as such; rather, it defines what strings the tool will output for each valid value.

.. code-block:: c

 
  { /* ENUM('yes','no') */
        name: "my_enum_column",
        type: FT_ENUM,
        fixed_length: 1,
        has_limits: TRUE,
        limits: {
                enum_values_count: 2,
                enum_values: { "yes", "no" }
        },
        can_be_null: FALSE
  },

Working with UTF8 and Other Multibyte Character Sets
====================================================

When multi-byte character sets such as UTF8 are used, InnoDB allocates more storage space for the value. For example,

 * ``CHAR(1)`` is stored as ``VARCHAR(3)``
 * ``VARCHAR(10)`` is stored as ``VARCHAR(30)``
 * ``VARCHAR(100)`` is stored as ``TEXT``

The table definitions need to be adjusted accordingly. Currently :file:`create_defs.pl` doesn't handle this automatically.

Using Multiple Table Definitions
================================

The ``constraints_parser`` tool actually supports more than one table definition at a time. You can place an array of table definitions into the structure of :file:`include/table_defs.h`. We have shown only one table at a time so far to keep things simple.

Doing this can help avoid frequent compilation, and can enable you to run the tool against a whole tablespace with many different tables' pages mixed together, but if some tables have the same structure, then the tool will not be able to tell the difference between them. That is why we suggest working one at a time.

If you do compile multiple table definitions into the tool at a time, then the output of the tool will naturally contain rows from different tables mingled together. You can use the :file:`split_dump.pl` tool to split the ``constraints_parser`` output into separate files.
