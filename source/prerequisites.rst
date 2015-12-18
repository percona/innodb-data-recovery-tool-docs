.. _prerequisites:

=============
Prerequisites
=============

It is important to understand that these tools are not designed for connecting to a live database and recovering data from it. They work on an offline copy of the data, and it is not safe to use them on a copy that is in use by |MySQL|. When you discover data loss or corruption, you should make it a priority to stop any further writing to the data files (or, if the data files have been lost, the disks themselves), and make a copy of the files to perform the recovery work on. You can do this by stopping |MySQL| and copying the files, or taking a filesystem snapshot with your *SAN* or *LVM* or similar. Do not copy the |InnoDB| files while |MySQL| is running; this is not safe.

In order to accomplish the data recovery, you will need the column names and data types in the tables to be recovered. The easiest way to obtain this is through ``SHOW CREATE TABLE``, although we will show several alternatives later. It is handy to have a working |MySQL| server with a backup of your database, even if it is quite old or the tables are empty, to help expedite the process. It is not necessary, though.

