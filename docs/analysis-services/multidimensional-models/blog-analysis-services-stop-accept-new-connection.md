---
title: Analysis Services stops accepting new connections –processing commit locks hurt | Microsoft Docs
description: Analysis Services stops accepting new connections –processing commit locks hurt
author: ramakoni1
ms.author: Jason.Howell
ms.reviewer: MashaMSFT
ms.service: backup-restore
ms.topic: conceptual blog
ms.date: 05/06/2019
---

# Analysis Services Stops Accepting New Connections

> [!NOTE]
> This article is derived from an MSDN blog originally posted on 07/03/2012.

When you process an Analysis Services object, such as FullProcess on a database or cube, old files need to be replaced with new files near the end of processing phase. In addition, a lock is needed for the highest level in the database. The users who run queries have the priority until the queries are finished.

Sometimes, the users and the server administrator cannot even sign in with management studio to run a new query.

An Analysis Services database is a collection of files (some XML files that point to files, and some compressed binary files) that represent the objects in the cube that you query with MDX queries. Processing is the act of refreshing those objects by using a new set of data values from the relational database. It runs large relational database Transact-SQL queries to query from the data sources, do joins, aggregate the data, save the compressed and aggregated data into the cube. The old copy of the AS database objects stays until the very end of the processing. When it is almost done, then the commit phase begins.

The commit phase needs an exclusive write lock, and the users can’t query the objects at that moment when it does the swap of the old version of the cube data for the new version.

Another problem is that the instance-wide Master.vmp lock is required to finish the commit from processing. This special file is a pointer to all the other database objects and their current versions, and this file is important when you swap out the old database objects with the new database objects.

As the server enters phase 2 of the commit, it tries to obtain a server level lock to update Master.vmp. If there is another transaction in process at that point, the server will wait for an interval that is equal to the ForceCommitTimeout setting (default is 30 seconds). Then, it will roll back any uncommitted transactions and abort executing queries. That server wide lock remains in effect until the transaction is finished and will block any read lock request that is initiated. When a new login or existing user tries to connect to the server, they will start a read lock request and wait.

This small file is the central point of the list of databases in Analysis Services, and should never be tampered with, or else your database is likely to be deleted.

![image](media/analysis-services-stops-accepting-new-connections/image1.png)

Insides of the master.vmp (shown with XML formatting for clarity) shows each object (represented by a GUID value) and the version number (an integer 1 2 3… 43, and so on). The version number gets incremented every time when the object is processed (or synchronized) and committed by the server, and the time is updated. This is the central point of all objects in an Analysis Services instance.

![image](media/analysis-services-stops-accepting-new-connections/image2.png)

## Why Can’t You Log in When Locking Happens

Locking can be the center of the problem. Here is a visual simplification of the blocking chain that prevents new users from getting into the database and running any query.

![image](media/analysis-services-stops-accepting-new-connections/image3.png)

You may encounter this locking pattern. Slow queries aggravate the processing commit waits and the server becomes unresponsive.
The head queries in Set 1 are taking many hours, and the Set 2 locks are waiting for more than one hour.

Set 3 New Logins >>> Set 2 Processing Commit phase >>> Set 1 Queries

- Set 1 Queries running holds database read locks (running for several hours)
- Set 2 Processing Commit needs commit write locks (waiting about 1 hour or more)
- Set 3 New connections wait in line, blocked to read the soon to be committed database

Sometimes the administrator cannot even sign in with management studio because the connection gets queued in Set 3.

When most new connections come in from Management Studio, the server does their initialization to see database names and object names with discover commands. They may get stuck in line and wait to read the soon-to-be committed and processed database behind the processing Set 2.

The new connections will likely do a discovery command as follows:

`Discover on DBSCHEMA_CATALOGS`

`Discover on MDSCHEMA_MEMBERS`

During the commit phase of the processing transaction, queries can still be sent to the object, but they will be queued until the commit is completed. For more information about locking and unlocking during processing, see [Processing Analysis Services Objects](https://docs.microsoft.com/previous-versions/sql/sql-server-2008/ms174860(v=sql.100)).

## How To Fix The Problem

### Step 1: Minimize MDX Query Duration

Tune the queries. Reduce the time that is required for Set 1 to finish. Then, you will have the least conflict between queries and processing. In one example, the slow query was requesting an arbitrary shape. Tune or Avoid Arbitrary Shape queries in Set 1 to run faster, change syntax to avoid arbitrary shapes, or configure a time-out in the application to kill long-running queries.

Add aggregations and partitions to reduce the amount of reading of data required.

Tune any calculations that may cause the formula engine a long time to work on.

Run a profiler trace to investigate the MDX query.

However, sometimes you cannot control the queries at all. In an ad hoc environment, when Excel pivot table users and custom MDX are enabled, you will have the occasional runaway MDX query that may just take a long time.

### Step 2: Avoid Processing at Peak Hours to Avoid Query and Processing Collisions

In one example, the Set 2 Full Processing happens at 11:30 a.m. and noon. There is bound to be a locking collision during those busy times because there are significant queries running in the business then. Avoid processing at peak times.

### Step 3: Tell Server to Favor One or The Other When Blocking Occurs

Try these two configuration settings to enable the server to try killing either the long queries Set 1 or the waiting processing Set 2.

- Kill the queries Set 2 can influence the server to cancel Set 1 after a time of waiting on locks with this setting:

  - ForceCommitTimeout is a server property that is used to control what happens when a processing operation is waiting to finish its operation to enter the commit phase. When this value is greater than zero, SSAS starts canceling prior transactions, but only after the specified value in milliseconds. However, if read locks become available before the ForceCommitTimeout period is reached, canceling will not occur.
- Kill the Processing Set 1 can influence the server to cancel Set 2 after waiting on locks occurs.

  - CommitTimeout Analysis Server processing operations need to acquire a write lock before it can commit a transaction. In order to acquire a write lock, no other read locks can be taken by another process or query. Therefore, Analysis Services needs to wait until all read locks are released. The transaction will wait for a while to acquire a write lock, as specified by the CommitTimeout property before rolling back.

Sometimes the cancellation doesn’t occur immediately, so even with the ForceCommitTimeout and CommitTimeout, there can be a limbo period where work is stalled.

## Another Variation – Multiple Processing Requests Can Block Each Other

If you run two or more processing batches at the same time in different transactions, a similar locking chain and deadlock may occur. This is too simplified, but just for illustration, assume there are two processing transactions that are ready near the same time, but get blocked waiting on a user’s long MDX query.

![image](media/analysis-services-stops-accepting-new-connections/image4.png)

The locking granularity at the end of processing is coarse at the database level and at the master.vmp file, so it is difficult to get parallel processing to go through successfully.

For Processing Start time is not as important as end time, so aim to avoid overlap in the end of processing jobs. If the two processing transactions are ready to commit around the same time, they may hold some locks and request other locks that cause a deadlock.

Adding a long-running MDX query into the mix makes is more likely to cause deadlock chain, because the intermediate locks can cause a circle to occur.

You might receive this error as a Notification event in the profiler trace:

```/
Transaction errors: Aborting transaction on session <victimsessionid>
```

The victim processing job will likely be canceled with this error:

```/
Transaction errors: While attempting to acquire one or more locks, the transaction was canceled.
```

## Proposed Remedies for Locking Conflict between Processing Jobs

1. Schedule processing in a staggered manner – remember that the end time is more important than the start time, because the commit phase is the time where the high granularity commit locks are needed.
2. Combine processing into a single transaction/batch XMLA tag.
If you process objects in the scope of a single transaction, maybe they won’t collide and kill each other. You can process parallel objects in a single transaction, instead of a sequence of small commits. You could have a larger more granular commit to reduce the window in which locks occur, but you are increasing the surface area of the number of locks at lower level granularity. This may increase conflict with user queries.
For example, you can have multiple processing commands in a single batch.

   ```xml
   <Batch xmlns="http://schemas.microsoft.com/analysisservices/2003/engine">
     <Parallel>
       <Process xmlns:xsd="http://www.w3.org/2001/XMLSchema"    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:ddl2="http://schemas.microsoft.com/analysisservices/2003/engine/2" xmlns:ddl2_2="http://schemas.microsoft.com/analysisservices/2003/engine/2/2" xmlns:ddl100_100="http://schemas.microsoft.com/analysisservices/2008/engine/100/100" xmlns:ddl200="http://schemas.microsoft.com/analysisservices/2010/engine/200" xmlns:ddl200_200="http://schemas.microsoft.com/analysisservices/2010/engine/200/200" xmlns:ddl300="http://schemas.microsoft.com/analysisservices/2011/engine/300" xmlns:ddl300_300="http://schemas.microsoft.com/analysisservices/2011/engine/300/300">
     <Object>
   ```

   ```xml
        <DatabaseID>AdventureWorksDW2012</DatabaseID>
        <DimensionID>Dim Account</DimensionID>
      </Object>
      <Type>ProcessUpdate</Type>
      <WriteBackTableCreation>UseExisting</WriteBackTableCreation>
    </Process>
    <Process xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:ddl2="http://schemas.microsoft.com/analysisservices/2003/engine/2" xmlns:ddl2_2="http://schemas.microsoft.com/analysisservices/2003/engine/2/2" xmlns:ddl100_100="http://schemas.microsoft.com/analysisservices/2008/engine/100/100" xmlns:ddl200="http://schemas.microsoft.com/analysisservices/2010/engine/200" xmlns:ddl200_200="http://schemas.microsoft.com/analysisservices/2010/engine/200/200" xmlns:ddl300="http://schemas.microsoft.com/analysisservices/2011/engine/300" xmlns:ddl300_300="http://schemas.microsoft.com/analysisservices/2011/engine/300/300">
      <Object>
   ```

   ```xml
                <DatabaseID>AdventureWorksDW2012</DatabaseID>
                <DimensionID>Clustered Customers</DimensionID>
            </Object>
            <Type>ProcessFull</Type>
            <WriteBackTableCreation>UseExisting</WriteBackTableCreation>
        </Process>
     </Parallel>
   </Batch>
   ```
3. Process on one server, sync to another server to ensure these processes do not interfere with each other.

## How Do You See These Locks and Blocking Chains

Starting with SQL Server 2008 R2 Service Pack 1, there are some great profiler trace events that help see these locks. There are XML tags within the text of the trace events that show who’s waiting, and which locks are held. Collect a profiler trace with the ordinary events, but add these events to see who’s blocking whom and for how long, and on which objects are the locks conflicting.

The Locks Acquired event indicates when the transaction has obtained a batch of locks for the processing of the transaction. The Locks Released event indicates when the transaction has released a batch of locks that the transaction requested. This event also indicates the duration that the locks are held. The Lock Waiting event indicates when a transaction tries and waits in a queue to obtain a lock in a batch. This information is in the TextData column of those events. This information includes the following additional related data: 

- The transaction ID
- The LockList XML node
- The WaitList XML node
- The HoldList XML node

The Lock Acquired event and the Lock Released event contain the LockList information. The Lock Waiting event contains the LockList, WaitList, and HoldList information. 

###LockList
The LockList node contains the following information:
 
- Lock type
- Lock status 
- Object path of the object that is being requested
- Object ID

Note: The object path is reported without a namespace. The Lock Released event additionally contains the Duration property. The Duration property indicates the duration that the lock is held in milliseconds.


The following is an example of the LockList node:

````<LockList>
<Lock>
<Type>Read</Type>
<LockStatus>Acquired</LockStatus>
<Object><DatabaseID>AdventureWorks</DatabaseID></Object>
<ObjectID>asadfb-vfbvadr-ft3323-54235</ObjectID>
</Lock>
<Lock>
<Type>Read</Type>
<LockStatus>Waiting</LockStatus>
<Object><DatabaseID>FoodMart</DatabaseID><Object>
<ObjectID>asadfb-vfbvadr-ft3323-54235</ObjectID>
</Lock>
<Lock>
<Type>Read</Type>
<LockStatus>Requested</LockStatus>
<Object><DatabaseID>FoodMart</DatabaseID><Object>
<ObjectID>asadfb-vfbvadr-ft3323-54235</ObjectID>
</Lock>
</LockList>````
In this example, the transaction requests three locks, obtains one, and waits for the second lock.

###WaitList
The WaitList node lists the waiting transactions that are ahead of the current transaction. The following is an example of the WaitList node: 

````<WaitList>
     <Object><DatabaseID>FoodMart</DatabaseID><Object>
     <ObjectID>asadfb-vfbvadr-ft3323-54235</ObjectID>
     <Type>Read</Type>
     <Transaction>  
  <TransactionID>2342-3we-dsdf-sdf<TransactionID>
  <SPID>234</SPID>
  <Type>Write</Type>
     </Transaction>  
     <Transaction>  
  <TransactionID>2ger342-3rtee-dsdf-sdf<TransactionID>
  <SPID>222</SPID>
  <Type>Read</Type>
     </Transaction>  
</WaitList>````

###HoldList
The HoldList node lists transactions that hold a lock that the current transaction tries to obtain. The following is an example of the HoldList node: 

```<HoldList>
     <Object><DatabaseID>FoodMart</DatabaseID><Object>
     <ObjectID>asadfb-vfbvadr-ft3323-54235</ObjectID>
     <Type>Read</Type>
     <Transaction>  
  <TransactionID>2342-3we-dsdf-sdf<TransactionID>
  <SPID>234</SPID>
  <Type>Write</Type>
     </Transaction>  
     <Transaction>  
  <TransactionID>2ger342-3rtee-dsdf-sdf<TransactionID>
  <SPID>222</SPID>
  <Type>Read</Type>
     </Transaction>  
</HoldList>```

In SQL Server 2008 Analysis Services or later versions, you can run an MDX query the dynamic management views to see the various connections, their transactions, and who’s granted locks and who’s waiting on locks (blocking).

```sql
select * from $system.discover_connections;
go
select * from $system.discover_sessions;
go
select * from $system.discover_transactions; 
go
select * from $system.discover_locks;
go
select * from $system.discover_jobs
go
```

## More Information

- [SQL Server Best Practices Article](https://docs.microsoft.com/previous-versions/sql/sql-server-2005/administrator/cc966525(v=technet.10))
- [SQL Server 2008 White Paper: Analysis Services Performance Guide](https://www.microsoft.com/download/details.aspx?id=17303)
