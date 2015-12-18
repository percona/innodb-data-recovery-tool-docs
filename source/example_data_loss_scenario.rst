.. _example_data_loss_scenario:

==========================
Example Data Loss Scenario
==========================

In this manual, we will show many examples based on the following data loss scenario. If you wish to follow along with the examples, you can create your own "sample data loss" too:

 1) Configure your server with :variable:`innodb_file_per_table` set to ``0``.
 2) Download and install the `Sakila sample database <http://dev.mysql.com/doc/sakila/en/sakila.html>`_
 3) Execute the following commands:

    .. code-block:: mysql 

      SET foreign_key_checks=0; TRUNCATE TABLE customer;

Here is a sample of the data in the table, which we will be seeing again in the future:

.. code-block:: mysql 

  mysql> SELECT * FROM customer LIMIT 6;
  +-------------+----------+------------+-----------+-------------------------------------+------------+--------+---------------------+---------------------+
  | customer_id | store_id | first_name | last_name | email                               | address_id | active | create_date         | last_update         |
  +-------------+----------+------------+-----------+-------------------------------------+------------+--------+---------------------+---------------------+
  |           1 |        1 | MARY       | SMITH     | MARY.SMITH@sakilacustomer.org       |          5 |      1 | 2006-02-14 22:04:36 | 2006-02-15 04:57:20 | 
  |           2 |        1 | PATRICIA   | JOHNSON   | PATRICIA.JOHNSON@sakilacustomer.org |          6 |      1 | 2006-02-14 22:04:36 | 2006-02-15 04:57:20 | 
  |           3 |        1 | LINDA      | WILLIAMS  | LINDA.WILLIAMS@sakilacustomer.org   |          7 |      1 | 2006-02-14 22:04:36 | 2006-02-15 04:57:20 | 
  |           4 |        2 | BARBARA    | JONES     | BARBARA.JONES@sakilacustomer.org    |          8 |      1 | 2006-02-14 22:04:36 | 2006-02-15 04:57:20 | 
  |           5 |        1 | ELIZABETH  | BROWN     | ELIZABETH.BROWN@sakilacustomer.org  |          9 |      1 | 2006-02-14 22:04:36 | 2006-02-15 04:57:20 | 
  |           6 |        2 | JENNIFER   | DAVIS     | JENNIFER.DAVIS@sakilacustomer.org   |         10 |      1 | 2006-02-14 22:04:36 | 2006-02-15 04:57:20 | 
  +-------------+----------+------------+-----------+-------------------------------------+------------+--------+---------------------+---------------------+
