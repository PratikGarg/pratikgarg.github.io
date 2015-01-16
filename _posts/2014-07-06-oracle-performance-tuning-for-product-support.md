---
layout: post
header-img: img/default-blog-pic.jpg
author: PGarg
description: 
post_id: 18523
created: 2014/07/06 20:05:07
created_gmt: 2014/07/06 15:05:07
comment_status: open
---

# Oracle Performance Tuning for Product Support

In the last post [SQL Server Performance Tuning for Product Support][1] we discussed about SQL Server tuning. In this blog I will cover Oracle (11g) database support in terms of performance and space.

**Performance Tuning** Compared to the SQL Server database we did not had much issues with Oracle database.

**Problem** As described in the [previous blog][2] about the application, its a sequence of steps. Where each step does some processing over the data stored in the database. Now some of these phases update certain table from empty to couple of million rows in little time. As a result of which the statistics information [(More about Table stats and Optimizer)][3] present with the database for tables become sort of invalid.

Now those tables which were empty and suddenly they have million of rows participate in queries as a join with other tables,  Indexes of those tables may or may not be used and the database may prefers to use full table scan because of earlier generated optimization plans when the table had little or no rows.

Even if the AUTO GENERATE stats are set to true the auto generated stats may not reflect the correct state. So you will need some mechanism to update stats either using a scheduled job or something else.  **Solution** In our case each steps updates a LOG TABLE whenever it starts and ends. So we created an AFTER INSERT TRIGGER on that table as given below. This trigger update the statistics of the database whenever a phase starts so the the query optimizer makes the appropriate adjustments based on current state of data.

`CREATE TRIGGER TRIGGERNAME AFTER INSERT ON LOGTABLE REFERENCING NEW AS N FOR EACH ROW BEGIN run_stats('TABLENAME'); END;`

` CREATE OR REPLACE PROCEDURE run_stats( p_table_name in varchar2 default null ) IS PRAGMA AUTONOMOUS_TRANSACTION; cv sys_refcursor; -- cursor variable v_table varchar2(30); v_owner varchar2(30); v_sql varchar2(2014) := 'select table_name from user_tables'; BEGIN IF( p_table_name is not null ) THEN v_sql := v_sql || ' where table_name = upper( ''' || p_table_name || ''') ' ; END IF; select sys_context( 'USERENV', 'CURRENT_USER' ) into v_owner from dual; open cv for v_sql; loop fetch cv into v_table; exit when cv%notfound; dbms_stats.gather_table_stats( v_owner, v_table, cascade=>true, method_opt=>'for all columns, size 1' ); end loop; close cv; end run_stats; ` **Space Tuning** **Problem** Our application was taking double the amount of space on Oracle as compared to other databases (DB2/SQLServer) for the same amount of data processed. Application has tables that have couple of regular fields and a CLOB datatype. Below is a sample table we have in application. Table can be  created it with two ways, each of which should give the same behavior.

` CREATE TABLE DEMO( a number (10, 2), data CLOB )`

` CREATE TABLE DEMO( a number (10, 2), data CLOB ) LOB (data) Stored AS (STORAGE IN ROW ENABLED) ` As per the oracle documentation when the CLOB is greater the 4000 bytes it will be stored outline else inline.

When the data is stored in this table for a clob value say "Hello" , the segment information for the "Demo table" and "Demo table LOB segment", shows that all the data is going to table and no new blocks are being consumed in the Lob Segment. When bigger is stored with total character less than 1500, then also the same behavior as above.

But when we store a data with total character > 2000 and < 3000 , then the LOB data is going to the LOB segment even though total character are less than 3000.

Now we couldn't understand Why the data smaller than 3000 characters is going to the LOB Segment ? . Is that each character takes 2 bytes , which justifies that data till 1500 is going to the data instead of Log Segment because of this Lots of disk space is getting wasted because of the LOB Table , since the CHUNK size is 8kb and the data per block will always be around 3 - 4K character and in some cases exceeding that. So essential for each row 4Kb space is wasted and in our case of 20 mn rows , it was running in 50's of GBs. Queries that you can use to see the disk usage per segment and per lob

`select segment_name,blocks from dba_segments where owner = 'SchemaName' order by blocks desc; select * from all_lobs where owner = 'SchemaName' `

**Solution** Solution is use compress the LOB data using High Compression.Initially we were skeptical with this setting because this may have big impact on the performance but of time required to compress/uncompress data during CRUD operations but surprisingly it did not acted that way.

` CREATE TABLE DEMO( a number (10, 2), data CLOB ) LOB (data) STORE AS SECUREFILE ( COMPRESS HIGH TABLESPACE USERS -- SHOULD BE SEPARATE TABLESPACE IF POSSIBLE ENABLE STORAGE IN ROW CHUNK 8192 RETENTION NOCACHE ) PCTFREE 0 NOLOGGING ; `

**After the above changes to the required table the space usage went drastically down and the total time degradation was negligible.**

**Others** **Using Composite indexes** As explained in the [previous blog][2] also, Use composite CLUSTERED/NON CLUSTERED index in you application to fasten the read operations.

**Dynamic Sampling** Set dynamic sampling in Oracle by default to level 4. This setting helps Oracle to pick the best possible plan when no statistics are present for the tables. Oracle will try to pick 64 blocks of records and extrapolate it to choose the right plan. This setting should be set by default and would help mainly for initial first run when no stats are present. `ALTER SYSTEM SET OPTIMIZER_DYNAMIC_SAMPLING = 4`

**Tools Used During Debugging** We used oracle Enterprise Manager for quite useful information. Please see some of the screen shots below.

To Check the explain plain of slow running queries go to **Home --> Performance --> Top Activity --> Select Query --> Plan --> Select Table Radio Option** ![wrw][4]

To check the size of current datafiles go to **Home --> Server -> DataFiles** ![Untitled][5]

To Check the current running queries go to **Home --> Performance --> SQL Monitoring** ![Untitled2][6]

   [1]: http://xebee.xebia.in/index.php/2014/07/03/sql-server-performance-tuning-for-product-support/
   [2]: http://xebee.xebia.in/index.php/2014/07/03/sql-server-performance-tuning-for-product-support/ (previous blog)
   [3]: http://www.dba-oracle.com/concepts/tables_optimizer_statistics.htm (More about Table stats and Optimizer)
   [4]: http://xebee.xebia.in/wp-content/uploads/2014/07/wrw.jpg
   [5]: http://xebee.xebia.in/wp-content/uploads/2014/07/Untitled.jpg
   [6]: http://xebee.xebia.in/wp-content/uploads/2014/07/Untitled2.jpg