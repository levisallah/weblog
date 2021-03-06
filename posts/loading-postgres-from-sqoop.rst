.. title: Loading Postgres from Sqoop
.. slug: loading-postgres-from-sqoop
.. date: 2018-11-11 21:19:55 UTC+01:00
.. tags: postgresql, sqoop, hadoop, big-data
.. category: data engineering
.. link: 
.. description: 
.. type: text

PostgreSQL and Hadoop are complementary technologies. PostgreSQL outperforms
Hadoop to retrieve small amount of information from multiple tables. Hadoop
outperforms PostgreSQL to transform multiple tables into simplfied and
aggregated tables. So, how to load PostgreSQL efficiently from Hadoop ?

.. END_TEASER

Sqoop as a bridge
=================

Hadoop stores data on its distributed filesystem: HDFS. At the end of the day,
the resulting procecced data is stored into a hdfs table and may need to be
loaded into a PosgreSQL table.

Copy is a efficient a robust PostgreSQL bulkloading function allowing to load
among from others CSV files. It is way faster than a bunch of parallel prepared
insert statements. 

ORC is a efficient and compressed Hadoop data format. It is both faster to
build and to read compared to CSV files.

As the bridge for Hadoop and relational databases, Sqoop manages all ORC, CSV,
COPY and INSERT methodologies. When dealing with PostgreSQL, there is no
perfect solution. 

Sqoop best approach
===================

There is two different scenario to load PostgreSQL.

:structured columns: Sqoop does not allow to use COPY together with ORC tables
                     but only with CSV. It splits them and run multiple COPY
                     statements from each part in parallel from HDFS. The CSV
                     format needs to be defined in the sqoop parameters. 


:unstructured columns: Text information may contain new lines. In this
                       situation, Sqoop is stuck when it splits the CSV into
                       multiple parts. Yet it is not possible to use the sqoop
                       direct mode, and the parallel inserts is the only way to
                       succeed. By chance, it is not necessar to transform the
                       result as CSV, and it is finally possible to exploit ORC
                       tables as-is.

Writing this post makes me thinking about a potential workaround to load
textual CSV from COPY by using only one mapper in this situation. Since COPY is
highly optimized it is possible that it won't degrade the performances while
making the overall process totally robust.


EDIT:

Indeed, using only one mapper allows to handle multiline CSV. Since COPY is
very fast and has been optimized to import dataset into postgreSQL, in the
general case, the method can be generalized to any kind of CSV to work in any
situation. That's the ultimate way to load PostgreSQL from Hadoop:

.. code-block:: bash

   sqoop export\
   --connect "jdbc:postgresql://<postgres_host>/<postgres_db>"\
   --username <postgres_user>\
   --password-file file:///home/$USER/.password\
   --export-dir /apps/hive/warehouse/<my_db>/<my_csv_table>\
   --table <my_postgres_table>\
   --columns "id, text_column"\
   -m 1\
   --direct\
   --lines-terminated-by '\n'\
   --fields-terminated-by ','\
   --input-null-string "\\\\N"\
   --input-null-non-string "\\\\N"\
   -- --schema <my_schema>


In order to prepare the table ready to be loaded by sqoop, it can be
transformed with a hive sql query:

.. code-block:: sql

   CREATE TABLE <my_db>.<my_csv_table> 
   STORED AS textfile  
   ROW FORMAT DELIMITED
   FIELDS TERMINATED BY ','
   LINES TERMINATED BY '\n'
   AS
   SELECT *
   FROM <my_hive_table>
