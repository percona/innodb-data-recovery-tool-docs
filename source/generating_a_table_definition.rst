.. _generating_a_table_definition:

=============================
Generating a Table Definition
=============================

Now that we know which pages contain our desired data, we need to find out the table's structure, create a table definition, compile it into the ``constraints_parser`` tool, and use that tool to extract the rows from the pages.

The table definintion includes the columns, column order, and data types. If the server is still running and the table has not been dropped, then we can use ``SHOW CREATE TABLE`` to gather this information. We will use this to create the table definition as a C struct that we will compile into the ``constraints_parser`` tool, which will extract the rows from the pages. The C struct is placed into the file :file:`include/table_defs.h`.

If possible, the easiest way to create the table definition is with the :program:`create_defs.pl` *Perl* script. It connects to the |MySQL| server and reads ``SHOW CREATE TABLE`` output, and prints the generated definition to its standard output. Here is an example:

.. code-block:: bash

  $ ./create_defs.pl --host=localhost --user=root --password=s3cret \
     --db=sakila --table=customer > include/table_defs.h 

In our example, the table structure we're working with is as follows:

.. code-block:: mysql

  CREATE TABLE `customer` (
    `customer_id` smallint(5) UNSIGNED NOT NULL AUTO_INCREMENT,
    `store_id` tinyint(3) UNSIGNED NOT NULL,
    `first_name` varchar(45) NOT NULL,
    `last_name` varchar(45) NOT NULL,
    `email` varchar(50) DEFAULT NULL,
    `address_id` smallint(5) UNSIGNED NOT NULL,
    `active` tinyint(1) NOT NULL DEFAULT '1',
    `create_date` datetime NOT NULL,
    `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY  (`customer_id`),
    KEY `idx_fk_store_id` (`store_id`),
    KEY `idx_fk_address_id` (`address_id`),
    KEY `idx_last_name` (`last_name`),
    CONSTRAINT `fk_customer_address` FOREIGN KEY (`address_id`) REFERENCES `address` (`address_id`) ON UPDATE CASCADE,
    CONSTRAINT `fk_customer_store` FOREIGN KEY (`store_id`) REFERENCES `store` (`store_id`) ON UPDATE CASCADE
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8

Here is the generated table definition that results.

.. code-block:: c

  #ifndef table_defs_h
  #define table_defs_h
  // Table definitions
  table_def_t table_definitions[] = {
        {
                name: "customer",
                {
                        { /* smallint(5) unsigned */
                                name: "customer_id",
                                type: FT_UINT,
                                fixed_length: 2,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        uint_min_val: 0,
                                        uint_max_val: 65535
                                },
                                can_be_null: FALSE
                        },
                        { /* Innodb's internally used field */
                                name: "DB_TRX_ID",
                                type: FT_INTERNAL,
                                fixed_length: 6,
                                can_be_null: FALSE
                        },
                        { /* Innodb's internally used field */
                                name: "DB_ROLL_PTR",
                                type: FT_INTERNAL,
                                fixed_length: 7,
                                can_be_null: FALSE
                        },
                        { /* tinyint(3) unsigned */
                                name: "store_id",
                                type: FT_UINT,
                                fixed_length: 1,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        uint_min_val: 0,
                                        uint_max_val: 255
                                },
                                can_be_null: FALSE
                        },
                        { /* varchar(45) */
                                name: "first_name",
                                type: FT_CHAR,
                                min_length: 0,
                                max_length: 45,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        char_min_len: 0,
                                        char_max_len: 45,
                                        char_ascii_only: TRUE
                                },
                                can_be_null: FALSE
                        },
                        { /* varchar(45) */
                                name: "last_name",
                                type: FT_CHAR,
                                min_length: 0,
                                max_length: 45,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        char_min_len: 0,
                                        char_max_len: 45,
                                        char_ascii_only: TRUE
                                },
                                can_be_null: FALSE
                        },
                        { /* varchar(50) */
                                name: "email",
                                type: FT_CHAR,
                                min_length: 0,
                                max_length: 50,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: TRUE,
                                        char_min_len: 0,
                                        char_max_len: 50,
                                        char_ascii_only: TRUE
                                },
                                can_be_null: TRUE
                        },
                        { /* smallint(5) unsigned */
                                name: "address_id",
                                type: FT_UINT,
                                fixed_length: 2,
                                has_limits: TRUE,
                                limits: {
                                        can_be_null: FALSE,
                                        uint_min_val: 0,
                                        uint_max_val: 65535
                                },
                                can_be_null: FALSE
                        },
                        { /* tinyint(1) */
                                name: "active",
                                type: FT_INT,
                                fixed_length: 1,
                                can_be_null: FALSE
                        },
                        { /* datetime */
                                name: "create_date",
                                type: FT_DATETIME,
                                fixed_length: 8,
                                can_be_null: FALSE
                        },
                        { /* timestamp */
                                name: "last_update",
                                type: FT_UINT,
                                fixed_length: 4,
                                can_be_null: FALSE
                        },
                        { type: FT_NONE }
                }
        },
  };
  #endif

There are a couple of special internal |InnoDB| columns in the definition. These are required for the tool to recognize the rows:

 * **DB_TRX_ID**: This column contains the ID of the last transaction to write this record.
 * **DB_ROLL_PTR**: This column contains a pointer to an undo log record where a previous version of the row is stored.
 * **DB_ROW_ID**: This should be the first field in tables without primary keys. Such tables have a special invisible primary key called ``GEN_CLUST_INDEX``, and this is the 6-byte auto-increment column contained in that primary key. Note that tables without an explicit primary key might not be built on a ``GEN_CLUST_INDEX`` index; if there is a non-nullable unique index in the table, |InnoDB| will choose that as its primary key in the absence of a primary key, and only create a ``GEN_CLUST_INDEX`` if there is nothing else suitable.

Rebuilding The Constraints Parser
=================================

You might need to edit the :file:`include/table_defs.h` file later, to fine-tune constraints on the data you are trying to recover. In the meantime, you need to rebuild the tool with *make*, so it is aware of your table definition.

.. code-block:: bash

  $ make
  gcc -DHAVE_OFFSET64_T -D_FILE_OFFSET_BITS=64 -D_LARGEFILE64_SOURCE=1 -D_LARGEFILE_SOURCE=1 -g -I include -I mysql-source/include -I mysql-source/innobase/include -c tables_dict.c -o lib/tables_dict.o
  gcc -DHAVE_OFFSET64_T -D_FILE_OFFSET_BITS=64 -D_LARGEFILE64_SOURCE=1 -D_LARGEFILE_SOURCE=1 -g -I include -I mysql-source/include -I mysql-source/innobase/include -o constraints_parser constraints_parser.c lib/tables_dict.o lib/print_data.o lib/check_data.o lib/libut.a lib/libmystrings.a
  gcc -DHAVE_OFFSET64_T -D_FILE_OFFSET_BITS=64 -D_LARGEFILE64_SOURCE=1 -D_LARGEFILE_SOURCE=1 -g -I include -I mysql-source/include -I mysql-source/innobase/include -o page_parser page_parser.c lib/tables_dict.o lib/libut.a

The tool is now ready to use.
