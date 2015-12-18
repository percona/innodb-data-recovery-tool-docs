.. _intro:

===================
MySQL Data Recovery
===================

This document will show an example of how to perform data recovery on a single |InnoDB| table. Before we begin, let us define the scope and purpose of these tools.

 * The tools work only for |InnoDB|/|XtraDB| tables, and will not work with |MyISAM| tables. Percona does have a preliminary set of tools for |MyISAM| data recovery; contact us if you need help with |MyISAM| data recovery.

 * The tools work on a saved copy of your data files, not on the running |MySQL| server.

 * There is no guarantee. Even with these tools, data is sometimes unrecoverable. For example, data that is overwritten cannot be recovered with these tools. There may be file system specific or physical means to recover overwritten data. This is outside the scope of this project.

 * Time is of the essence. The best chance for recovery comes when you act immediately to save a copy of your raw data files as soon as you discover the loss or corruption.

 * There is manual work to do. Not everything is automatic.

 * Recovery depends on knowing your data. As part of the process you may have to choose between two versions of your data. The better you know your data, the better the chance you'll be able to recover it.

You can reverse several types of data loss with these tools, including:

 * A mistaken ``DROP TABLE``, ``DELETE``, ``TRUNCATE``, or ``UPDATE``

 * Deletion of the data file on the filesystem level

 * Partial corruption of the data such that |InnoDB| is unable to run without crashing, even with :variable:`innodb_force_recovery` set


