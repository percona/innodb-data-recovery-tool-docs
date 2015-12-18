.. _advanced_recovery_techniques:

============================
Advanced Recovery Techniques
============================

This page contains various script snippets and helpful techniques for recovering data from difficult cases.

Finding Index IDs
=================

The following script will parse the output of the |InnoDB| table monitor, and print every table and the index ID of its ``PRIMARY`` index:

``find-index-id.sh`` :

.. code-block:: bash

  #!/bin/sh
  # Invoke this script with the name of the file containing the tablespace info
  awk '
     $1 == "TABLE:" {
        n = substr($3, 1, length($3) -1);
     }
     $1 == "INDEX:" && $3 ~ /PRIMARY|GEN_CLUST_INDEX/ {
        i = substr($3, 1, length($3) -1);
        printf "%-30s %-15s %d %d\n", n, i, $5, substr($6, 1, length($6) - 1);
     }' "$@"

It often happens that you don't know the index for a given table when you recover data, perhaps because you are working from a raw device or a completely unbootable data file. There are several ways to find out which index ID contains the pages you are looking for.

The simplest way is perhaps with :program:`grep`. If there are ``VARCHARS`` or ``CHARS`` in the table and you know possible values, you can :program:`grep` the whole tablespace to find your index id. If you find several different indexes that contain the string you are looking for, try different strings and see if you can narrow it down. Here is an example:

.. code-block:: bash

  $ grep -r "XXX-yyyy-ZzZz" pages-1217332715/
  Binary file pages-1217332715/0-276/1116-00001116.page matches
  Binary file pages-1217332715/0-276/596-00000596.page matches
  Binary file pages-1217332715/0-276/715-00000715.page matches
  Binary file pages-1217332715/0-276/1119-00001119.page matches
  Binary file pages-1217332715/0-276/1237-00001237.page matches
  Binary file pages-1217332715/0-276/1101-00001101.page matches
  Binary file pages-1217332715/0-276/709-00000709.page matches

If you are lucky, you'll find only one index ID by grepping a tablespace based on a known string. If not, then a longer string will probably give a better match. If you know there is a John Smith in the table, and that the first-name and last-name columns are stored adjacent to each other in that order, then you can try to grep for two strings concatenated together, e.g. ``grep -r JohnSmith pages-*``. This will work because there is no ``\0`` after John in the row format; column values are not stored null-terminated.

If you are you completely unaware what is in the table, or there are only numeric columns such as ``INT`` or ``DATE``, it is more difficult to find matching pages, but still sometimes possible.

If you have no strings to search for, but you know some numbers such as primary key values, you can use binary :program:`grep` (:program:`bgrep`) to find pages containing the values. Standard :program:`grep` cannot match binary values, only string values. More information on this technique is in a `blog post by Aurimas Mikalauskas <https://www.percona.com/blog/2010/11/09/lost-innodb-tables-xfs-and-binary-grep/>`_, one of Percona's data recovery specialists.

Another way is to compile the ``constraints_parser`` tool with the table definition you are looking for. Make sure you define only one table. Then run the script below. The idea is that if the page contains rows matching the table structure you are looking for, then constraints_parser will find some records.

``find-pages-with-rows.sh``:

.. code-block:: bash

  #!/bin/sh
  # Execute this script with one argument: the directory containing your page files.
  find "$@" -name '*.page' | while read file; do
     num_found=$(./constraints_parser -5 -f "$file" | wc -l)
     if [ $num_found -gt 0 ]
     then
        echo $num_found $file
     fi
  done

Determining the Page Format
===========================

If you don't know what format a page is in, the ``constraints_parser`` tool can tell you. Run the tool in verbose mode (``-V``), and if your guess was wrong, it will tell you:

 * “Page is in COMPACT format while we're looking for REDUNDANT - skipping” or
 * “Page is in REDUNDANT format while we're looking for COMPACT - skipping”

You can also run the following. It will output 0 for REDUNDANT pages (version 4), and 1 for COMPACT (version 5) pages.

.. code-block:: bash

  dc -e "2o `hexdump --Cd /path/to/page/file | grep 00000020 | awk '{ print $12}'` p" | sed 's/./& /g' | awk '{ print $1}'

Recovering From a Raw Partition
===============================

The tools do not need to be used on files in a filesystem. If you unmount the filesystem and point the ``page_parser`` tool at the raw device, it can scan it just like a file, and split out the pages. It identifies pages by looking for the internal ``infimum`` and ``supremum`` records, so it is fairly good at finding pages, and is quite fast.

Debugging Match Failures
========================

If the ``constraints_parser`` tool finds nothing but you see your data in the files, it is useful to run it in verbose mode. Let's say you have found a string “abcd” in a page. Your :program:`table_defs.h` is:

.. code-block:: c

  { /* int(11) */
        name: "id",
        type: FT_INT,
        fixed_length: 4,
        has_limits: TRUE,
        limits: {
                can_be_null: FALSE,
                int_min_val: 1,
                int_max_val: 100000
        },
        can_be_null: FALSE
  },
  { /*  */
        name: "DB_TRX_ID",
        type: FT_INTERNAL,
        fixed_length: 6,
        can_be_null: FALSE
  },
  { /*  */
        name: "DB_ROLL_PTR",
        type: FT_INTERNAL,
        fixed_length: 7,
        can_be_null: FALSE
  },
  { /* varchar(100) */
        name: "name",
        type: FT_CHAR,
        min_length: 0,
        max_length: 100,
        has_limits: TRUE,
        limits: {
                can_be_null: TRUE,
                char_min_len: 0,
                char_max_len: 100,
                char_ascii_only: TRUE
                },
        can_be_null: TRUE
  },

Each row will contain 4+6+7 = 17 bytes before the “abcd” string (this is the size of the preceding columns in the row). The place where the row's data begins is called the ``ORIGIN``. Find the origin in a hex editor, then look for the offset equal to the origin in the ``constraints_parser`` verbose output. This should help you see why your record isn't found.

A Script for Recovering Many Tables
===================================

If there are many tables to recover, this script may help you. Execute it with the database name you want to recover.

``recover-tables.sh``:

.. code-block:: bash

  #!/bin/sh
   
  db=$1
   
  tables=`mysql -ss -u root -e "SHOW TABLES" $db`
   
  for i in $tables
  do
        #Check how many rows has a table
        rows=`mysql -u root -e "SELECT COUNT(*) FROM $i" -s $db`
        if [ $rows -ne 0 ]
        then
                # Prepare environment
                echo "Restoring table $i"
                table=$i
                cd include && rm -f table_defs.h && ln -s table_defs.h.$table table_defs.h
                cd ..
                make clean all
                # Restoring rows
                found=0
                while [ $found -lt 1 ]
                do
                        echo ""
                        read -p "Enter the path to directory where data of table $i might be: " dir
                        cat $dir/*.page > p
                        ./constraints_parser -5 -f p >> out.$i
                        found=`cat out.$i | wc -l`
                done
        fi
  done

Recovering The Data Dictionary
==============================

|InnoDB|'s internal data dictionary is stored in system tables, which are tables like any other. If you cannot boot up |InnoDB| to find index IDs, then perhaps you can simply extract the data from the system tables and find the table names and index IDs you are looking for. There are two tables of interest:

* ``SYS_INDEXES``, with index ID 0-3, and the following table structure:

  .. code-block:: mysql

    CREATE TABLE `SYS_INDEXES` (
      `TABLE_ID` bigint(20) UNSIGNED NOT NULL DEFAULT '0',
      `ID` bigint(20) UNSIGNED NOT NULL DEFAULT '0',
      `NAME` varchar(120) DEFAULT NULL,
      `N_FIELDS` int(10) UNSIGNED DEFAULT NULL,
      `TYPE` int(10) UNSIGNED DEFAULT NULL,
      `SPACE` int(10) UNSIGNED DEFAULT NULL,
      `PAGE_NO` int(10) UNSIGNED DEFAULT NULL,
       PRIMARY KEY  (`TABLE_ID`,`ID`)
     ) ENGINE=InnoDB DEFAULT CHARSET=latin1

* ``SYS_TABLES``, with index ID 0-1, and the following table structure:

  .. code-block :: mysql

    CREATE TABLE `SYS_TABLES` (
      `NAME` varchar(255) NOT NULL DEFAULT '',
      `ID` bigint(20) UNSIGNED NOT NULL DEFAULT '0',
      `N_COLS` int(10) UNSIGNED DEFAULT NULL,
      `TYPE` int(10) UNSIGNED DEFAULT NULL,
      `MIX_ID` bigint(20) UNSIGNED DEFAULT NULL,
      `MIX_LEN` int(10) UNSIGNED DEFAULT NULL,
      `CLUSTER_NAME` varchar(255) DEFAULT NULL,
      `SPACE` int(10) UNSIGNED DEFAULT NULL,
       PRIMARY KEY  (`NAME`)
     ) ENGINE=InnoDB DEFAULT CHARSET=latin1

Both tables are in REDUNDANT format. The following two code sections contain the table definitions for these system tables:

``sys_indexes_table_defs.h``:

.. code-block:: c

  #ifndef table_defs_h
  #define table_defs_h
   
  // Table definitions
  table_def_t table_definitions[] = {
        {
                name: "SYS_INDEXES",
                {
                        { /* bigint(20) unsigned */
                                name: "TABLE_ID",
                                type: FT_UINT,
                                fixed_length: 8,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        uint_min_val: 0,
                                        uint_max_val: 18446744073709551615ULL
                                },
                                can_be_null: FALSE
                        },
                        { /* bigint(20) unsigned */
                                name: "ID",
                                type: FT_UINT,
                                fixed_length: 8,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        uint_min_val: 0,
                                        uint_max_val: 18446744073709551615ULL
                                },
                                can_be_null: FALSE
                        },
                        { /*  */
                                name: "DB_TRX_ID",
                                type: FT_INTERNAL,
                                fixed_length: 6,
                                can_be_null: FALSE
                        },
                        { /*  */
                                name: "DB_ROLL_PTR",
                                type: FT_INTERNAL,
                                fixed_length: 7,
                                can_be_null: FALSE
                        },
                        { /* varchar(120) */
                                name: "NAME",
                                type: FT_CHAR,
                                min_length: 0,
                                max_length: 120,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: TRUE,
                                        char_min_len: 0,
                                        char_max_len: 120,
                                        char_ascii_only: TRUE
                                },
                                can_be_null: TRUE
                        },
                        { /* int(10) unsigned */
                                name: "N_FIELDS",
                                type: FT_UINT,
                                fixed_length: 4,
                                can_be_null: TRUE
                        },
                        { /* int(10) unsigned */
                                name: "TYPE",
                                type: FT_UINT,
                                fixed_length: 4,
                                can_be_null: TRUE
                        },
                        { /* int(10) unsigned */
                                name: "SPACE",
                                type: FT_UINT,
                                fixed_length: 4,
                                can_be_null: TRUE
                        },
                        { /* int(10) unsigned */
                                name: "PAGE_NO",
                                type: FT_UINT,
                                fixed_length: 4,
                                can_be_null: TRUE
                        },
                        { type: FT_NONE }
                }
        },
  };
                                                 
  #endif

``sys_tables_table_defs.h``:

.. code-block:: c

  #ifndef table_defs_h
  #define table_defs_h
   
  // Table definitions
  table_def_t table_definitions[] = {
        {
                name: "SYS_TABLES",
                {
                        { /* varchar(255) */
                                name: "NAME",
                                type: FT_CHAR,
                                min_length: 0,
                                max_length: 255,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        char_min_len: 0,
                                        char_max_len: 255,
                                        char_ascii_only: TRUE
                                },
                                can_be_null: FALSE
                        },
                        { /*  */
                                name: "DB_TRX_ID",
                                type: FT_INTERNAL,
                                fixed_length: 6,
                                can_be_null: FALSE
                        },
                        { /*  */
                                name: "DB_ROLL_PTR",
                                type: FT_INTERNAL,
                                fixed_length: 7,
                                can_be_null: FALSE
                        },
                        { /* bigint(20) unsigned */
                                name: "ID",
                                type: FT_UINT,
                                fixed_length: 8,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        uint_min_val: 0,
                                        uint_max_val: 18446744073709551615ULL
                                },
                                can_be_null: FALSE
                        },
                        { /* int(10) unsigned */
                                name: "N_COLS",
                                type: FT_UINT,
                                fixed_length: 4,
                               can_be_null: TRUE
                        },
                        { /* int(10) unsigned */
                                name: "TYPE",
                                type: FT_UINT,
                                fixed_length: 4,
                                can_be_null: TRUE
                        },
                        { /* bigint(20) unsigned */
                                name: "MIX_ID",
                                type: FT_UINT,
                                fixed_length: 8,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: TRUE,
                                        uint_min_val: 0,
                                        uint_max_val: 18446744073709551615ULL
                                },
                                can_be_null: TRUE
                        },
                        { /* int(10) unsigned */
                                name: "MIX_LEN",
                                type: FT_UINT,
                                fixed_length: 4,
                                can_be_null: TRUE
                        },
                        { /* varchar(255) */
                                name: "CLUSTER_NAME",
                                type: FT_CHAR,
                                min_length: 0,
                                max_length: 255,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: TRUE,
                                        char_min_len: 0,
                                        char_max_len: 255,
                                        char_ascii_only: TRUE
                                },
                                can_be_null: TRUE
                        },
                        { /* int(10) unsigned */
                                name: "SPACE",
                                type: FT_UINT,
                                fixed_length: 4,
                                can_be_null: TRUE
                        },
                        { type: FT_NONE }
                }
        },
  };

  #endif

Getting CREATE TABLE From .frm Files
====================================

If you don't have the ``CREATE TABLE`` definitions for the tables you're trying to recover, but you have the :file:`.frm` files, it's possible to get the ``CREATE TABLE`` from them. Boot up a |MySQL| server (any server will do – preferably some sandbox or other non-production instance), and create an |InnoDB| table with the desired name. The structure does not matter at all. Shut down the server, and overwrite the :file:`.frm` file in the data directory with the :file:`.frm` file from the table you're trying to recover. Start |MySQL| again, and you should be able to run ``SHOW CREATE TABLE`` and get the definition from the :file:`.frm` file.

Recovering Data From Secondary Indexes
======================================

Usually you need only the primary key to recover data. However, if there are other indexes in a table, and some of the primary key blocks are unrecoverable, then you can try to recover some data from them. An |InnoDB| page is the same format regardless of the index. The non-primary index pages contain tuples with the key columns, followed by the primary key columns. The index ID can be found in ``SYS_TABLES/SYS_INDEXES``, or in the output of the tablespace monitor, as usual. All you have to do is to prepare the :file:`include/table_defs.h` as usual, but with the index's columns instead of the primary key columns. Omit the special internal columns (``DB_DRX_ID``, ``DB_ROLL_PTR``), but for tables with no primary key, use ``DB_ROW_ID`` as the primary key column.

Here is an example. Given the following table,

.. code-block:: mysql

  CREATE TABLE `message_comments` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `message_id` int(11) NOT NULL DEFAULT '0',
    `author_id` int(11) NOT NULL DEFAULT '0',
    `body` text,
    `created_at` datetime DEFAULT NULL,
    `updated_at` datetime DEFAULT NULL,
    `attachments_count` int(11) NOT NULL DEFAULT '0',
   PRIMARY KEY (`id`),
   KEY `index_message_comments_on_message_id_and_created_at` (`message_id`,`created_at`)
   ) ENGINE=InnoDB AUTO_INCREMENT=33133 DEFAULT CHARSET=latin1

You need the following contents in :file:`include/table_defs.h`:

.. code-block:: c

  #ifndef table_defs_h
  #define table_defs_h
   
  // Table definitions
  table_def_t table_definitions[] = {
          {
                name: "index_message_comments_on_message_id_and_created_at",
                {
                        { /* int(11) */
                                name: "message_id",
                                type: FT_INT,
                                fixed_length: 4,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        int_min_val: -2147483648LL,
                                        int_max_val: 2147483647LL
                                },
                                can_be_null: FALSE
                        },
                        { /* datetime */
                                name: "created_at",
                                type: FT_DATETIME,
                                fixed_length: 8,
                                can_be_null: TRUE
                        },
                        { /* int(11) */
                                name: "id",
                                type: FT_INT,
                                fixed_length: 4,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        int_min_val: -2147483648LL,
                                        int_max_val: 2147483647LL
                                },
                                can_be_null: FALSE
                        },
                        { type: FT_NONE }
                }
        },
  };

  #endif
