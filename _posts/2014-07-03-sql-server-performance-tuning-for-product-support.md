---
layout: post
author: PGarg
description: 
post_id: 18472
created: 2014/07/03 10:03:09
created_gmt: 2014/07/03 05:03:09
comment_status: open
tags: SQL Server Performance Tuning
image: /assets/article_images/2014-07-06-oracle/desktop.jpg
---

# SQL Server Performance Tuning for Product Support

**Overview** For last few months I have been working with a client for developing a product based on Java landscape. As a product it needs to support couple of databases at the back end depending upon the client's customers requirement and by support I mean database should meet certain benchmarks in terms of time performance and space usage. So I along with couple of more folks have been working on this implementation. Team did not had much expertise in database so issues took bit longer than usual to fade away. In this blog I would like to share some problems that we faced and how we resolved them.

**Overview of Application** Application is a pipeline of steps/phases, where first phase takes input as a file (.txt | .csv)  having 1 to mn records, stores them in the database and then the next subsequent steps do processing over that stored dataand final step produces a new file containing with transformed records. The transformations performed on the data interesting only in terms of business, so not going into those details. Some of the phases make heavy use of Multithreading to process non overlapped data to increase the performance and better utilize the CPU Cores. The application already supports **DB2** database and the complete processing finishes with in a given time and space.

**Technologies Used in Application**

  * Java & Apache Camel for the pipeline described above.
  * Apache MetaModel (Incubator) as ORM
**Performance Tuning** Team had least exposure to this database and the reason being this database rarely finds its place in the Java world but anyways we had to support it.  **Initial Test** So starting with few records pushed to the pipeline. There were some errors in processing but mostly due to the SQL syntax errors or datatype mismatch but nonetheless the complete processing did happen successfully.

**First Blocker** Next step was to run the pipeline with a bigger data set. The processing got suspended in one of the phases. Fired the below query to see queries blocking each other. ` SELECT sqltext.TEXT,req.session_id,req.status, req.command,req.cpu_time,req.total_elapsed_time, sqlplan.query_plan,W.* FROM sys.dm_exec_requests req CROSS APPLY sys.dm_exec_sql_text(sql_handle) AS sqltext CROSS APPLY sys.dm_exec_query_plan(plan_handle) AS sqlplan left join sys.dm_os_waiting_tasks W on W.session_id = req.session_id `

Figured out that a long running SELECT Query on TABLE A is blocking the smaller UPDATE queries on the same TABLE A . The SELECT query (a long running transaction) is an iterator over records to fetch X records each time from the database and then those X fetched records were divided and processed separately by threads which fire UPDATE query on the same table. This results in the suspension of UPDATE query.

**Solution** **So changing the transaction level to READ_COMMITTED_SNAPSHOT ON  to resolve this issue. **More details about this issue can be found here <http://msdn.microsoft.com/en-us/library/tcbchxcb(v=vs.110).aspx>

**Second Blocker** Some of the phases in the execution pipeline were really slow. So first thing that comes to mind is about the indexes not being correctly implemented. To verify that we need to see the explain plan of queries. SQL server comes with an Activity Monitor Tool. This tool gives quite useful information separated across sections . Please see snapshot below

![Screenshot from 2014-07-02 20:06:01][1]

![Stats][2]

Select the query from the long running transactions and see the execution plan by right clicking on it. Now the execution plan doesn't tells much about the slowness of the query though it will give the relative percentages of the execution time. So to actually see how much this query is taking time. Select the same query from the **processes section** and right click to analyse in profiler. There we can find out the quantified time for the query. The profiler showed us that each query is taking .5 sec to 1 sec to execute which is way too slow than expected.  Again looked at  index scripts and found out that both the clustered and non clustered indexes were more or less applied correctly as per the guidelines mentioned. Below are some of those guidelines

**Clustered Index :-** This Index defines how the data is stored and sorted in the table based on a particular key. Can be only one per table. Generally created using Primary Key or Candidate Keys. Also please note the **FILLFACTOR=80**. This setting may end up taking more space but will speed up the index operations. ` CREATE UNIQUE CLUSTERED INDEX [IDX] ON [DBO].[TABLE] ( [COL1] ASC, [COL2] ASC )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = OFF, FILLFACTOR=80) ON [PRIMARY]; ` **Non Clustered Index :- **Can be more than one in the table, Should be created on the columns which appear in the queries and also use composite index on all the keys of the same table that appear in a query. ` CREATE NONCLUSTERED INDEX [IDX] ON [DBO].[TABLE] ( [Col1] ASC, [Col2] ASC, [Col3] ASC )WITH (SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF,ALLOW_PAGE_LOCKS = OFF, ALLOW_ROW_LOCKS= ON) ON [PRIMARY] ` So even after loads of permutation and combination for indexes we got no performance improvement. Which probably means indexes were already correct. We let the processing run with whatever time its taking.

**Third Blocker** Next we ran into deadlock issues in of the phases. This phases involved multiple threads from the application adding/updating/deleting rows across the **same and different tables** but none of the threads overlaps with the rows to be updated in the tables. So it appeared that the queries are taking table level locks on indexes/tables. To avoid that we updated the tables and indexes with below settings. With these setting we still got the deadlock so apparently the issue was something else.

` ALTER INDEX ... WITH (ALLOW_PAGE_LOCKS = OFF, ALLOW_ROW_LOCKS = ON); ALTER TABLE SET LOCK_ESCALATION OFF `

Also profiled the queries (**Go to Tool -> SQL Server Profile -> Start New Trace -> Show All Events -> Select All Deadlock Events**). Though there were deadlock Events but no lock escalation.

![m7AKT][3]

**Solution **

   [1]: http://xebee.xebia.in/wp-content/uploads/2014/07/Screenshot-from-2014-07-02-200601.png
   [2]: http://xebee.xebia.in/wp-content/uploads/2014/07/Stats.jpg
   [3]: http://xebee.xebia.in/wp-content/uploads/2014/07/m7AKT.png