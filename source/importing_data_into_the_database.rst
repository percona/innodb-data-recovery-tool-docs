.. _importing_data_into_the_database:

================================
Importing Data Into the Database
================================

To complete the data recovery, you can load the output of the constraints_parser tool into the database with ``LOAD DATA INFILE``. At this point we are done using the tools; the process from here on is all performed with the database, or with other tools such as Unix command-line utilities.

To import the file with ``LOAD DATA INFILE``, you can use the following rules:

 * The file is tab delimited. Use ``FIELDS TERMINATED BY '\t'``.
 * Columns are quoted. Use ``OPTIONALLY ENCLOSED BY '"'``.
 * Each line starts with the table name and a tab. Use ``LINES STARTING BY '<table_name>\t'``.
 * ``TIMESTAMP`` values are stored as integer values. Use ``FROM_UNIXTIME(@var)``.
 * Use ``REPLACE INTO`` when you have old row versions that you want to ignore and overwrite with the latest version.

To complete our running example, we can execute the following SQL:

.. code-block:: mysql

  LOAD DATA INFILE '/tmp/customer_data.tsv'
  REPLACE INTO TABLE customer
  FIELDS TERMINATED BY '\t'
  OPTIONALLY ENCLOSED BY '"'
  LINES STARTING BY 'customer\t'
  (customer_id, store_id, first_name, last_name, email,
     address_id, active, create_date, @last_update)
  SET last_update = FROM_UNIXTIME(@last_update);


