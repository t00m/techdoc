---
Category: Note
Command: sqlplus
Database: Oracle
Date: 2026-05-06
DocType: Reference
Filesystem: /oracle
OS: Linux, Unix
Priority: Normal
Product: AIX, SQLPlus, RHEL
Project: SAP Basis Maintenance, Database Maintenance
Public: Yes
Scope: Database Administration
Status: Released
Table: ALL_TABLES, DATABASE_PROPERTIES, DBA_DATA_FILES, DBA_FREE_SPACE, DBA_HIST_SEG_STAT, DBA_HIST_SNAPSHOT, DBA_INDEXES, DBA_OBJECTS, DBA_REGISTRY, DBA_SEGMENTS, DBA_TABLES, DBA_TABLESPACES, DBA_TEMP_FILES, DBA_TEMP_FREE_SPACE, DBA_USERS, RSSTATMANREQMAP, SYS.V_$TEMP_EXTENT_POOL, SYS.V_$TEMP_SPACE_HEADER, T000, USER_SEGMENTS, USR02, V$DATABASE, V$DATAFILE, V$INSTANCE, V$LOG, V$LOGFILE, V$PARAMETER, V$SESSION_CONNECT_INFO, V$VERSION
Tag: autoextend, datafile, discovery, growth, tracking, locking, segment, undo, group, instance, query, redolog, size, table, tablespace
Team: SAP
Topic: Administration, Archiving, Housekeeping, Maintenance, Monitoring, Performance, User
User: oracle
---

# Useful Oracle queries for SAP Basis Administrators

## Overview

This is a comprehensive reference of SQL*Plus queries for SAP Basis Administrators working with Oracle databases. It covers database administration tasks including instance and database monitoring, tablespace and datafile management, table and index analysis, user and security management, and growth trend analysis using Oracle’s Automatic Workload Repository (AWR).

All queries should work in any Oracle 10g+ instance with SAP schema (`SAPSR3` owner). Queries use standard Oracle data dictionary views and dynamic performance views.

*DBSID* is `SDB`.

## Prerequisites and Setup

* SQL*Plus access to Oracle database with DBA or monitoring privileges
* Knowledge of basic SQL syntax and Oracle object naming conventions
* AWR enabled (for historical segment statistics queries)
* SAP tables populated (for SAP-specific queries)

## Configuration

Before running queries, apply these SQL*Plus settings to control output formatting:

```sql
SET LINESIZE 32767
SET WRAP OFF
SET PAGESIZE 100
SET LINESIZE 150
SPOOL <filename>    -- Optional: redirect output to file
-- ... run queries ...
SPOOL OFF
```

For detailed formatting options, see the [SQL*Plus User’s Guide and Reference](http://docs.oracle.com/cd/B19306_01/server.102/b14357/ch12040.htm).

### Common Format Variables

These column formatting statements are used throughout this guide:

```sql
COL SEGMENT_NAME FORMAT A20
COL SEGMENT_TYPE FORMAT A15
COL SIZE_KB FORMAT 999,999,990.99
COL SIZE_MB FORMAT 999,999,990.99
COL SIZE_GB FORMAT 99,990.99
COL TABLESPACE_NAME FORMAT A32
COL OWNER FORMAT A15
COL OBJECT_NAME FORMAT A36
COL TABLE_NAME FORMAT A30
COL USERNAME FORMAT A15
COL ACCOUNT_STATUS FORMAT A10
COL AUTHENTICATION_TYPE FORMAT A15
```

## Database

### Show Oracle Database version

```sql
SELECT
FROM v$version;
```

**Output**

```
BANNER
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 – 64bit Production
PL/SQL Release 11.2.0.3.0 – Production CORE    11.2.0.3.0      Production
TNS for IBM/AIX RISC System/6000: Version 11.2.0.3.0 – Production
NLSRTL Version 11.2.0.3.0 – Production
```

### Show database details

```sql
SELECT NAME, LOG_MODE, OPEN_MODE, DATABASE_ROLE, PLATFORM_NAME
FROM  v$database;
```

**Output**

```
NAME LOG_MODE     OPEN_MODE   DATABASE_ROLE PLATFORM_NAME
SDB  NOARCHIVELOG READ WRITE  PRIMARY       AIX-Based Systems (64-bit)
```

### Show database size

```sql
SELECT SUM(BYTES)/1024/1024/1024 AS "DBSIZE(GB)"
FROM dba_data_files;
```

**Output**

```
DBSIZE(GB)
5312.59766
```

### Show Oracle Instant Client Version

```sql
SELECT DISTINCT client_version
FROM v$session_connect_info
WHERE sid = sys_context('userenv', 'sid');
```

**Output**

```
CLIENT_VERSION
11.2.0.3.0
```

For a list of all possibilities to check and identify Oracle Instant Client Version check this [document](http://scn.sap.com/docs/DOC-61410).

#### Installed components in Database
```sql
SELECT comp_name, version, status
FROM dba_registry;
```

## Instance

### Show database instance details

```sql
SELECT INSTANCE_NAME, HOST_NAME, VERSION, STARTUP_TIME, STATUS, INSTANCE_ROLE
FROM v$instance;
```

**Output**

```
INSTANCE_NAME HOST_NAME VERSION    STARTUP_TIME STATUS INSTANCE_ROLE
SDB           SAPSERVER 11.2.0.3.0 26-MAR-15    OPEN   PRIMARY_INSTANCE
```

### Monitor instance status of oracle

```sql
SELECT host_name, status
FROM v$instance;
```

### DB Schema for SDB
```sql
SELECT OWNER
FROM DBA_TABLES WHERE TABLE_NAME = 'T000';
```

**Output**

```
OWNER
SAPSR3
```

## Tablespaces

### Show tablespaces details

```sql
SELECT TABLESPACE_NAME, STATUS, CONTENTS, SEGMENT_SPACE_MANAGEMENT
FROM dba_tablespaces;
```

**Output**

```
TABLESPACE_NAME                STATUS    CONTENTS  SEGMEN
SYSTEM                         ONLINE    PERMANENT MANUAL
PSAPUNDO                       ONLINE    UNDO      MANUAL
SYSAUX                         ONLINE    PERMANENT AUTO
PSAPTEMP                       ONLINE    TEMPORARY MANUAL
PSAPSR3                        ONLINE    PERMANENT AUTO
PSAPSR3USR                     ONLINE    PERMANENT AUTO
TOOLS                          ONLINE    PERMANENT AUTO
PSAPSR3731                     ONLINE    PERMANENT AUTO
```

### List of datafiles for tablespace

```sql
SELECT file_name
FROM dba_data_files
WHERE tablespace_name='<TABLESPACE_NAME>';
```

### Find tables being used by tablespace

```sql
SELECT table_name
FROM dba_tables
WHERE tablespace_name='PSAPSR37XX';
```

### Check autoextend

```sql
SELECT TABLESPACE_NAME,SEGMENT_SPACE_MANAGEMENT
FROM dba_tablespaces;
```

### Check what is the retention period for PSAPUNDO tablespace

```sql
SELECT name "retention",value/60 "minutes"
FROM v$parameter
WHERE name like '%undo_retention%';
```

### Get total, used and free space for tablespaces

```sql
set pages 49999 lin 120
col tablespace_name for a32 tru
col "Total GB"  for 999,999.9
col "GB Used"   for 999,999.9
col "GB Free"   for 99,999.9
col "Pct Free"  for 999.9
col "Pct Used"  for 999.9
comp sum of "Total GB"  on report
comp sum of "GB Used"   on report
comp sum of "GB Free"   on report
break on report

Select A.Tablespace_Name, B.Total/1024/1024/1024 "Total GB",
       (B.Total-a.Total_Free)/1024/1024/1024 "GB Used",
       A.Total_Free/1024/1024/1024 "GB Free",
       (A.Total_Free/B.Total) * 100 "Pct Free",
       ((B.Total-A.Total_Free)/B.Total) * 100 "Pct Used"
  From (Select Tablespace_Name, Sum(Bytes) Total_Free
          From Sys.Dba_Free_Space
         Group By Tablespace_Name     ) A
     , (Select Tablespace_Name, Sum(Bytes) Total
          From Sys.Dba_Data_Files
         Group By Tablespace_Name     ) B
Where A.Tablespace_Name = B.Tablespace_Name
Order By 1
/
```

### PSAPTEMP

Get freespace:

```sql
SQL> select tablespace_size/1024/1024 as tbsize, free_space/1024/1024 as free from dba_temp_free_space;

    TBSIZE       FREE
---------- ----------
    172000      48290
```

Get properties:

```sql
SQL>  column property_name format a25;
SQL> column property_value format a15;
SQL> SELECT property_name, property_value as value from database_properties WHERE property_name='DEFAULT_TEMP_TABLESPACE';

PROPERTY_NAME             VALUE
------------------------- ---------------
DEFAULT_TEMP_TABLESPACE   PSAPTEMP
```

Get details:

```sql
SQL> set linesize 150
SQL> column tablespace format a25;
SQL> column filename format a45;
SQL> SELECT tablespace_name as tablespace, file_name as filename, bytes/1024/1024 as MB, status from dba_temp_files;

TABLESPACE                FILENAME                                              MB STATUS
------------------------- --------------------------------------------- ---------- -------
PSAPTEMP                  /oracle/SDB/sapdata1/temp_4/temp.data4             20000 ONLINE
PSAPTEMP                  /oracle/SDB/sapdata1/temp_8/temp.data8             20000 ONLINE
PSAPTEMP                  /oracle/SDB/sapdata1/temp_9/temp.data9             32000 ONLINE
PSAPTEMP                  /oracle/SDB/sapdata1/temp_7/temp.data7             20000 ONLINE
PSAPTEMP                  /oracle/SDB/sapdata1/temp_2/temp.data2             20000 ONLINE
PSAPTEMP                  /oracle/SDB/sapdata1/temp_5/temp.data5             20000 ONLINE
PSAPTEMP                  /oracle/SDB/sapdata1/temp_3/temp.data3             20000 ONLINE
PSAPTEMP                  /oracle/SDB/sapdata1/temp_6/temp.data6             20000 ONLINE

8 rows selected.
```

**Recreate**:

1. Create a new temporary tablespace:

   ```sql
   $ mkdir /oracle/SDB/sapdata1/temp_0/
   sql> CREATE TEMPORARY TABLESPACE "PSAPTEMP123" TEMPFILE '/oracle/SDB/sapdata1/temp_0/temp.data123' SIZE 100M REUSE AUTOEXTEND OFF;
   ```
2. Make it default:

   ```
   ALTER DATABASE DEFAULT TEMPORARY TABLESPACE PSAPTEMP123;
   ```
3. Drop the old one

   ```
   DROP TABLESPACE PSAPTEMP INCLUDING CONTENTS;
   ```
4. Create another one like the original

   ```sql
   sql> create temporary tablespace PSAPTEMP tempfile '/oracle/SDB/sapdata1/temp_1/temp.data1' size 1000M autoextend on next 100M maxsize 32000M;
   sql> ALTER TABLESPACE PSAPTEMP ADD TEMPFILE '/oracle/SDB/sapdata1/temp_2/temp.data2' size 1000M autoextend on next 100M maxsize 32000M;
   sql> ALTER TABLESPACE PSAPTEMP ADD TEMPFILE '/oracle/SDB/sapdata1/temp_3/temp.data3' size 1000M autoextend on next 100M maxsize 32000M;
   sql> ALTER TABLESPACE PSAPTEMP ADD TEMPFILE '/oracle/SDB/sapdata1/temp_4/temp.data4' size 1000M autoextend on next 100M maxsize 32000M;
   sql> ALTER TABLESPACE PSAPTEMP ADD TEMPFILE '/oracle/SDB/sapdata1/temp_5/temp.data5' size 1000M autoextend on next 100M maxsize 32000M;
   sql> ALTER TABLESPACE PSAPTEMP ADD TEMPFILE '/oracle/SDB/sapdata1/temp_6/temp.data6' size 1000M autoextend on next 100M maxsize 32000M;
   sql> ALTER TABLESPACE PSAPTEMP ADD TEMPFILE '/oracle/SDB/sapdata1/temp_7/temp.data7' size 1000M autoextend on next 100M maxsize 32000M;
   sql> ALTER TABLESPACE PSAPTEMP ADD TEMPFILE '/oracle/SDB/sapdata1/temp_8/temp.data8' size 1000M autoextend on next 100M maxsize 32000M;
   ```
5. Make it default:

   ```
   ALTER DATABASE DEFAULT TEMPORARY TABLESPACE PSAPTEMP
   ```
6. Drop the first temporary tablespace

   ```
   DROP TABLESPACE PSAPTEMP123 INCLUDING CONTENTS;
   ```

If _ERROR ORA-60100: dropping temporary tablespace with tablespace ID number (tsn) 3 is blocked due to sort segments_, apply the following [solution](https://support.oracle.com/knowledge/Oracle%20Database%20Products/2696984_1.html):

```
shutdown immediate
startup;
DROP TABLESPACE PSAPTEMP INCLUDING CONTENTS;
```

Check PSAPTEMP freespace:

```
select tablespace_size/1024/1024 as tbsize, free_space/1024/1024 as free from dba_temp_free_space;
```

List datafiles in PSAPTEMP:

```
set linesize 150
column tablespace format a25;
column filename format a45;
SELECT tablespace_name as tablespace, file_name as filename, bytes/1024/1024 as MB, status from dba_temp_files;
```

## Tables

### Get table size

```sql
select segment_name,sum(bytes)/1024/1024/1024 GB from dba_segments where segment_type='TABLE' and segment_name=upper('&TABLE_NAME') group by segment_name;
```

## Users

### Show database users

```sql
SET PAGESIZE 100;
COLUMN USERNAME FORMAT A25;
COLUMN ACCOUNT_STATUS FORMAT A10;
COLUMN AUTHENTICATION_TYPE FORMAT A15;
SELECT USERNAME, ACCOUNT_STATUS, AUTHENTICATION_TYPE FROM dba_users;
```

**Output**

```
USERNAME                       ACCOUNT_STATUS                   AUTHENTI
SAPSR3                         OPEN                             PASSWORD
SYSTEM                         OPEN                             PASSWORD
SYS                            OPEN                             PASSWORD
OPS$SAPSERVICESAP              OPEN                             EXTERNAL
OPS$ORASAP                     OPEN                             EXTERNAL
```

### To see all user who were locked by the system admin

```sql
SELECT bname
FROM <schema-name>.USR02
WHERE uflag='64' and mandt='<client-id>';
```

### List of users

```sql
COLUMN USERNAME FORMAT A15;
COLUMN LAST_LOGIN FORMAT A50;
COLUMN ACCOUNT_STATUS FORMAT A10;
SELECT USERNAME, ACCOUNT_STATUS, LAST_LOGIN FROM DBA_USERS WHERE LAST_LOGIN IS NOT NULL ORDER BY LAST_LOGIN DESC;

USERNAME        ACCOUNT_ST LAST_LOGIN
--------------- ---------- --------------------------------------------------
SAPSR3          OPEN       20-NOV-23 09.15.58.000000000 AM +01:00
DBSNMP          OPEN       20-NOV-23 09.04.31.000000000 AM +01:00
ETLSAP          OPEN       20-NOV-23 05.03.38.000000000 AM +01:00
OPS$ORABWD      OPEN       20-NOV-23 04.00.50.000000000 AM +01:00
OPS$ORACLE      OPEN       19-NOV-23 07.00.45.000000000 AM +01:00
SYSTEM          OPEN       13-NOV-23 01.41.19.000000000 PM +01:00
OPS$BWDADM      OPEN       15-JUL-23 07.04.01.000000000 PM +01:00

7 rows selected.
```

### (Un)lock user account

```sql
ALTER USER <USERNAME> ACCOUNT LOCK;
```

```sql
ALTER USER <USERNAME> ACCOUNT UNLOCK;
```

### Create user

```sql
CREATE USER <USERNAME> IDENTIFIED BY <PASSWORD> [PROFILE <PROFILE_NAME>]
```

### Delete SAP* user (or another user)

First we check if user exists:

```sql
SELECT MANDT, BNAME
FROM <DB_SCHEMA>.USR02
WHERE MANDT = 'XXX' AND BNAME = 'SAP*';
```

Then delete it:

```sql
DELETE
FROM <DB_SCHEMA>.USR02
WHERE MANDT = 'XXX' AND BNAME = 'SAP*';
```

### Password complexity

Execute script sap_utlpwdmg.sql from SAP Note 1522952 in SQL Plus:

```sql
@sap_utlpwdmg.sql
```

### Activate complexity in profiles:

```sql
ALTER PROFILE DEFAULT LIMIT PASSWORD_VERIFY_FUNCTION verify_function_sap;
ALTER PROFILE SAPUPROF LIMIT PASSWORD_VERIFY_FUNCTION verify_function_sap;
```

### Deativate complexity in profiles:

```sql
ALTER PROFILE DEFAULT LIMIT PASSWORD_VERIFY_FUNCTION NULL;
ALTER PROFILE SAPUPROF LIMIT PASSWORD_VERIFY_FUNCTION NULL;
```

## Datafiles

### Check datafiles

```sql
SELECT FILE#, STATUS, ENABLED
FROM  v$datafile;
```

**Output**

```
     FILE# STATUS  ENABLED
       375 ONLINE  READ WRITE
       376 ONLINE  READ WRITE
       377 ONLINE  READ WRITE
       378 ONLINE  READ WRITE
       379 ONLINE  READ WRITE
       380 ONLINE  READ WRITE
```

### List of datafiles

```sql
SELECT FILE_NAME AS Datafile, BYTES/1024/1024 AS "Size(MB)"
FROM DBA_DATA_FILES
WHERE TABLESPACE_NAME LIKE '%<TABLESPACE_NAME>%';
```

**Output**

```
Datafile                                          Size(MB)
[…]
/oracle/SDB/sapdata9/sr3731_3/sr3731.data3           20000
/oracle/SDB/sapdata9/sr3731_4/sr3731.data4           20000
/oracle/SDB/sapdata9/sr3731_5/sr3731.data5           20000
/oracle/SDB/sapdata9/sr3731_6/sr3731.data6           20000
/oracle/SDB/sapdata9/sr3731_7/sr3731.data7            3500
/oracle/SDB/sapdata5/sr3_348/sr3.data348             31744
```

### Resize datafile until size

```sql
ALTER DATABASE DATAFILE '<PATH_TO_DATAFILE>' RESIZE <SIZE>M;
```

## Redologs

### List of redologs groups and files belonging to each group

```sql
SET LINESIZE 150
SET PAGESIZE 200
COLUMN GROUP FORMAT A10;
COLUMN MEMBER FORMAT A45;

SELECT a.group#, a.member, b.bytes/1024/1024 AS "SIZE(MB)"
FROM v$logfile a, v$log b
WHERE a.group# = b.group#;

```

**Output**

```
    GROUP# MEMBER                                          SIZE(MB)
---------- --------------------------------------------- ----------
         1 /oracle/SDB/origlogA/log_g11m1.dbf                   200
         1 /oracle/SDB/mirrlogA/log_g11m2.dbf                   200
         2 /oracle/SDB/origlogB/log_g12m1.dbf                   200
         2 /oracle/SDB/mirrlogB/log_g12m2.dbf                   200
         3 /oracle/SDB/origlogA/log_g13m1.dbf                   200
         3 /oracle/SDB/mirrlogA/log_g13m2.dbf                   200
         4 /oracle/SDB/origlogB/log_g14m1.dbf                   200
         4 /oracle/SDB/mirrlogB/log_g14m2.dbf                   200

8 rows selected.
```

### Active Redolog groups

```sql
SET LINESIZE 150
SET PAGESIZE 200
COLUMN GROUP FORMAT A10;
COLUMN MEMBER STATUS A20;
SELECT group#, status
FROM v$log;
```

**Output:**

```
    GROUP# STATUS
---------- ----------------
         1 INACTIVE
         2 INACTIVE
         3 CURRENT
         4 INACTIVE
```

### Check the online redolog files details

```sql
SET LINESIZE 150
SET PAGESIZE 200
COLUMN GROUP FORMAT A10;
COLUMN MEMBER FORMAT A45;

SELECT l.group#,f.member,l.archived,l.bytes/1024/1024  AS "SIZE(MB)",l.status,f.type
FROM v$log l, v$logfile f
WHERE l.group# = f.group#;
```

**Output:**

```
    GROUP# MEMBER                                   ARC      BYTES STATUS           TYPE
---------- ---------------------------------------- --- ---------- ---------------- -------
         1 /oracle/SDB/origlogA/log_g11m1.dbf       YES  194.43711 INACTIVE         ONLINE
         1 /oracle/SDB/mirrlogA/log_g11m2.dbf       YES  194.43711 INACTIVE         ONLINE
         2 /oracle/SDB/origlogB/log_g12m1.dbf       YES  194.43711 INACTIVE         ONLINE
         2 /oracle/SDB/mirrlogB/log_g12m2.dbf       YES  194.43711 INACTIVE         ONLINE
         3 /oracle/SDB/origlogA/log_g13m1.dbf       NO   194.43711 CURRENT          ONLINE
         3 /oracle/SDB/mirrlogA/log_g13m2.dbf       NO   194.43711 CURRENT          ONLINE
         4 /oracle/SDB/origlogB/log_g14m1.dbf       YES  194.43711 INACTIVE         ONLINE
         4 /oracle/SDB/mirrlogB/log_g14m2.dbf       YES  194.43711 INACTIVE         ONLINE

8 rows selected.
```

## Redologs

### List of redologs groups and files belonging to each group

```sql
SET LINESIZE 150
SET PAGESIZE 200
COLUMN GROUP FORMAT A10;
COLUMN MEMBER FORMAT A45;

SELECT a.group#, a.member, b.bytes/1024/1024 AS "SIZE(MB)"
FROM v$logfile a, v$log b
WHERE a.group# = b.group#;

```

**Output**

```
    GROUP# MEMBER                                          SIZE(MB)
---------- --------------------------------------------- ----------
         1 /oracle/SDB/origlogA/log_g11m1.dbf                   200
         1 /oracle/SDB/mirrlogA/log_g11m2.dbf                   200
         2 /oracle/SDB/origlogB/log_g12m1.dbf                   200
         2 /oracle/SDB/mirrlogB/log_g12m2.dbf                   200
         3 /oracle/SDB/origlogA/log_g13m1.dbf                   200
         3 /oracle/SDB/mirrlogA/log_g13m2.dbf                   200
         4 /oracle/SDB/origlogB/log_g14m1.dbf                   200
         4 /oracle/SDB/mirrlogB/log_g14m2.dbf                   200

8 rows selected.
```

### Active Redolog groups

```sql
SET LINESIZE 150
SET PAGESIZE 200
COLUMN GROUP FORMAT A10;
COLUMN MEMBER STATUS A20;
SELECT group#, status
FROM v$log;
```

**Output:**

```
    GROUP# STATUS
---------- ----------------
         1 INACTIVE
         2 INACTIVE
         3 CURRENT
         4 INACTIVE
```

### Check the online redolog files details

```sql
SET LINESIZE 150
SET PAGESIZE 200
COLUMN GROUP FORMAT A10;
COLUMN MEMBER FORMAT A45;

SELECT l.group#,f.member,l.archived,l.bytes/1024/1024  AS "SIZE(MB)",l.status,f.type
FROM v$log l, v$logfile f
WHERE l.group# = f.group#;
```

**Output:**

```
    GROUP# MEMBER                                   ARC      BYTES STATUS           TYPE
---------- ---------------------------------------- --- ---------- ---------------- -------
         1 /oracle/SDB/origlogA/log_g11m1.dbf       YES  194.43711 INACTIVE         ONLINE
         1 /oracle/SDB/mirrlogA/log_g11m2.dbf       YES  194.43711 INACTIVE         ONLINE
         2 /oracle/SDB/origlogB/log_g12m1.dbf       YES  194.43711 INACTIVE         ONLINE
         2 /oracle/SDB/mirrlogB/log_g12m2.dbf       YES  194.43711 INACTIVE         ONLINE
         3 /oracle/SDB/origlogA/log_g13m1.dbf       NO   194.43711 CURRENT          ONLINE
         3 /oracle/SDB/mirrlogA/log_g13m2.dbf       NO   194.43711 CURRENT          ONLINE
         4 /oracle/SDB/origlogB/log_g14m1.dbf       YES  194.43711 INACTIVE         ONLINE
         4 /oracle/SDB/mirrlogB/log_g14m2.dbf       YES  194.43711 INACTIVE         ONLINE

8 rows selected.
```

## Table Size Analysis

This section covers queries for analyzing individual table and index sizes in the SAP schema.

### Get single table size

Query the total allocated space (in GB) for a specific table:

```sql
SELECT segment_name,
       SUM(bytes)/1024/1024/1024 AS GB
FROM dba_segments
WHERE segment_type = 'TABLE'
  AND segment_name = UPPER('&TABLE_NAME')
GROUP BY segment_name;
```

**Parameters**

* `&TABLE_NAME` — table name (case-insensitive; will be converted to uppercase)

**Example**

To find the size of `RSSTATMANREQMAP`:

```sql
SELECT segment_name,
       SUM(bytes)/1024/1024/1024 AS GB
FROM dba_segments
WHERE segment_type = 'TABLE'
  AND segment_name = UPPER('RSSTATMANREQMAP')
GROUP BY segment_name;
```

### Get table size from USER_SEGMENTS

If you are logged in as the table owner, query `USER_SEGMENTS` instead of `DBA_SEGMENTS`:

```sql
SELECT segment_name,
       SUM(bytes)/1024/1024/1024 AS GB
FROM user_segments
WHERE segment_type = 'TABLE'
  AND segment_name = UPPER('SAPSR3.RSSTATMANREQMAP')
GROUP BY segment_name;
```


### Size multiple specific tables with their indexes

When you need to size a specific set of tables by name:

```sql
COL SEGMENT_NAME FORMAT A20
COL SEGMENT_TYPE FORMAT A15
COL SIZE_KB FORMAT 999,999,990.99
COL SIZE_MB FORMAT 999,999,990.99
COL SIZE_GB FORMAT 99,990.99
SET LINESIZE 200
SET PAGESIZE 100

SELECT 
    segment_name,
    segment_type,
    ROUND(bytes/1024, 2) AS size_kb,
    ROUND(bytes/1024/1024, 2) AS size_mb,
    ROUND(bytes/1024/1024/1024, 2) AS size_gb
FROM dba_segments
WHERE owner = 'SAPSR3'
AND (
    (segment_name IN ('<TABLE_1>', '<TABLE_2>') AND segment_type = 'TABLE')
    OR 
    (segment_name IN (
        SELECT index_name 
        FROM dba_indexes 
        WHERE table_owner = 'SAPSR3' 
        AND table_name IN ('<TABLE_1>', '<TABLE_2>')
    ) AND segment_type LIKE 'INDEX%')
)
ORDER BY segment_type, bytes DESC;
```

**Parameters**

* Modify the list in `segment_name IN (...)` to match your target tables.
* Results are ordered by type and size (largest first).

## Segment Growth and Historical Analysis

This section provides queries using Oracle’s Automatic Workload Repository (AWR) to track segment growth over time.

> **❗ IMPORTANT**\
AWR must be enabled and retention set appropriately. Growth queries require at least two snapshots spanning your target time period.

### Table size over last 30 days

Query historical segment usage data to see how a table grew:

```sql
SELECT 
    snap_id,
    TO_CHAR(begin_interval_time, 'YYYY-MM-DD') AS snap_date,
    tablespace_name,
    object_name,
    SUM(space_used_total) / 1024 / 1024 AS size_mb
FROM 
    dba_hist_seg_stat ss
    JOIN dba_hist_snapshot sn ON ss.snap_id = sn.snap_id
WHERE 
    object_name LIKE '%<PATTERN>%'
    AND owner = 'SAPSR3'
GROUP BY 
    snap_id, begin_interval_time, tablespace_name, object_name
ORDER BY 
    snap_date;
```

### Detailed growth tracking with object types

This query provides object type classification and is filtered to a specific date range:

```sql
COL SNAP_DATE FORMAT A10
COL OBJECT_NAME FORMAT A30
COL OBJECT_TYPE FORMAT A15
COL OWNER FORMAT A6
COL SIZE_KB FORMAT 999,999,990.99
COL SIZE_MB FORMAT 999,999,990.99
COL SIZE_GB FORMAT 99,990.99
SET LINESIZE 200
SET PAGESIZE 20000

SELECT 
    TO_CHAR(sn.begin_interval_time, 'YYYY-MM-DD') AS snap_date,
    o.owner,
    o.object_name,
    o.object_type,
    SUM(ss.space_used_total) / 1024 / 1024 AS size_mb
FROM 
    dba_hist_seg_stat ss
    JOIN dba_hist_snapshot sn ON ss.snap_id = sn.snap_id 
      AND ss.dbid = sn.dbid
    JOIN dba_objects o ON ss.obj# = o.object_id
WHERE 
    o.object_name LIKE '<TABLE_1>'
    AND o.owner = 'SAPSR3'
    AND o.object_type LIKE '%TABLE%'
    AND sn.begin_interval_time >= TRUNC(SYSDATE) - 30
    AND sn.begin_interval_time < TRUNC(SYSDATE) + 1
GROUP BY 
    TO_CHAR(sn.begin_interval_time, 'YYYY-MM-DD'),
    o.owner,    
    o.object_name,
    o.object_type
ORDER BY 
    snap_date;
```

**Notes**

* Change `TRUNC(SYSDATE) - 30` to adjust the lookback window.
* Include `object_type` to distinguish tables from their associated objects.

### Daily space delta (growth rate) with current total size

This advanced query shows the daily space delta for objects and correlates it with their current total size in `dba_segments`:

```sql
COLUMN owner                  FORMAT A16
COLUMN object_name            FORMAT A36
COLUMN start_day              FORMAT A11
COLUMN block_increase_mbytes  FORMAT 999,999,999.99
COLUMN total_size_mb          FORMAT 999,999,999.99
SET LINESIZE 200
SET PAGESIZE 20000

SPOOL CHANGES.txt 

SELECT
    obj.owner,
    obj.object_name,
    obj.object_type,
    TO_CHAR(sn.begin_interval_time, 'RRRR-MON-DD') AS start_day,
    SUM(a.space_used_delta) / 1024 / 1024 AS block_increase_mbytes,

    /* Current total size from DBA_SEGMENTS (MB) */
    (
        SELECT SUM(s.bytes) / 1024 / 1024
        FROM dba_segments s
        WHERE s.owner = obj.owner
          AND (
                s.segment_name = obj.object_name
             OR s.segment_name IN (
                    SELECT i.index_name
                    FROM dba_indexes i
                    WHERE i.table_owner = obj.owner
                      AND i.table_name  = obj.object_name
                )
          )
    ) AS total_size_mb

FROM
    dba_hist_seg_stat   a,
    dba_hist_snapshot   sn,
    dba_objects         obj
WHERE
    sn.snap_id = a.snap_id
    AND a.instance_number = sn.instance_number
    AND obj.object_id = a.obj#
    AND obj.object_type LIKE '%TABLE%'
    AND obj.owner NOT IN ('SYS','SYSTEM')
    AND sn.end_interval_time BETWEEN
        TO_TIMESTAMP('17-JAN-2026','DD-MON-RRRR')
        AND TO_TIMESTAMP('21-JAN-2026','DD-MON-RRRR')
GROUP BY
    obj.owner,
    obj.object_name,
    obj.object_type,
    TO_CHAR(sn.begin_interval_time,'RRRR-MON-DD')
ORDER BY
    obj.object_name,
    start_day
/

SPOOL OFF
```

**Parameters**

* `TO_TIMESTAMP('17-JAN-2026', 'DD-MON-RRRR')` — start of date range (adjust as needed)
* `TO_TIMESTAMP('21-JAN-2026', 'DD-MON-RRRR')` — end of date range (adjust as needed)

**Output Interpretation**

* `block_increase_mbytes` — daily growth (positive = growing, negative = shrinking)
* `total_size_mb` — current size from live dictionary (snapshot of today’s allocation)

### Simplified daily growth without current size lookup

For better performance, use this simplified version without the correlated subquery:

```sql
COL SNAP_DATE FORMAT A10
COL OBJECT_NAME FORMAT A30
COL OBJECT_TYPE FORMAT A15
COL OWNER FORMAT A6
COL SIZE_KB FORMAT 999,999,990.99
COL SIZE_MB FORMAT 999,999,990.99
COL SIZE_GB FORMAT 99,990.99
SET LINESIZE 200
SET PAGESIZE 20000

COLUMN owner FORMAT A16
COLUMN object_name FORMAT A36
COLUMN start_day FORMAT A11
COLUMN block_increase_mbytes FORMAT 999,999,999.99
SET LINESIZE 200
SET PAGESIZE 20000

SPOOL CHANGES.txt 

SELECT   
    obj.object_name, 
    obj.object_type,
    TO_CHAR(sn.BEGIN_INTERVAL_TIME,'RRRR-MON-DD') AS start_day,
    SUM(a.SPACE_USED_DELTA) / 1024 / 1024 AS block_increase_mbytes
FROM     
    dba_hist_seg_stat a,
    dba_hist_snapshot sn,
    dba_objects obj
WHERE    
    sn.snap_id = a.snap_id
    AND a.instance_number = sn.instance_number
    AND obj.object_id = a.obj#
    AND obj.object_type LIKE '%TABLE%'
    AND obj.owner NOT IN ('SYS','SYSTEM') 
    AND sn.end_interval_time BETWEEN 
        TO_TIMESTAMP('17-JAN-2026','DD-MON-RRRR')
        AND TO_TIMESTAMP('21-JAN-2026','DD-MON-RRRR')
GROUP BY 
    obj.owner, obj.object_name, obj.object_type,
    TO_CHAR(sn.BEGIN_INTERVAL_TIME,'RRRR-MON-DD')
ORDER BY 
    obj.object_name, start_day
/

SPOOL OFF
```

**Notes**

* Much faster than the correlated subquery version.
* Use when you only need daily growth rates, not current total size.

## SAP Queries

### System in upgrade — enable imports

When the system is marked as being in upgrade and you need to enable imports:

```sql
UPDATE SAPSR3.uvers SET PUTSTATUS='+';
COMMIT;
```

**⚠️ WARNING**\
Only execute this if you intend to allow imports during a system upgrade.

### BRBACKUP error — clear lock

If BRBACKUP reports it is currently running or was killed:

**Error message**

```
BR0051I BRBACKUP 6.40 (43)
BR0055I Start of database backup: bebchpaa.anf 2014-01-16 01.00.34
BR0484I BRBACKUP log file: /oracle/SDB/sapbackup/bebchpaa.anf
BR0071E BRBACKUP currently running or was killed
BR0072I Please delete file /oracle/SDB/sapbackup/.lock.brb if BRBACKUP was killed
BR0073E Setting of BRBACKUP lock failed
BR0056I End of database backup: bebchpaa.anf 2014-01-16 07.00.04
BR0280I BRBACKUP time stamp: 2009-07-26 07.00.05
BR0054I BRBACKUP terminated with errors
```

Connect to database:

```bash
sqlplus /nolog
SQL> connect /as sysdba
```

End backup:

```sql
ALTER DATABASE END BACKUP;
```

Then check if the lock file exists and delete it. If not found, run the backup again.

## Instance Management

### Show instance startup time

Query when the current instance was started:

```sql
SELECT instance_name, 
       TO_CHAR(startup_time, 'mm/dd/yyyy hh24:mi:ss') AS startup_time 
FROM v$instance;
```

**Output**

```
INSTANCE_NAME STARTUP_TIME
------------- ---------------------
SDB           01/26/2026 14:23:45
```

## Tablespace Capacity and Free Space

This section provides comprehensive queries for monitoring tablespace allocation, usage, and capacity.

### Detailed tablespace usage (permanent and temporary)

This complex query provides a unified view of both permanent and temporary tablespace usage:

```sql
SET LINESIZE 150

SELECT a.tablespace_name AS "Tablespace Name",
       ROUND(a.bytes_alloc / 1024 / 1024) AS "Allocated (MB)",
       ROUND(NVL(b.bytes_free, 0) / 1024 / 1024) AS "Free (MB)",
       ROUND((a.bytes_alloc - NVL(b.bytes_free, 0)) / 1024 / 1024) AS "Used (MB)",
       ROUND((NVL(b.bytes_free, 0) / a.bytes_alloc) * 100) AS "% Free",
       100 - ROUND((NVL(b.bytes_free, 0) / a.bytes_alloc) * 100) AS "% Used",
       ROUND(maxbytes / 1024 / 1024) AS "Max. Bytes (MB)"
FROM (
    SELECT f.tablespace_name,
           SUM(f.bytes) bytes_alloc,
           SUM(DECODE(f.autoextensible, 'YES', f.maxbytes, 'NO', f.bytes)) maxbytes
    FROM dba_data_files f
    GROUP BY tablespace_name
) a,
(
    SELECT f.tablespace_name,
           SUM(f.bytes) bytes_free
    FROM dba_free_space f
    GROUP BY tablespace_name
) b
WHERE a.tablespace_name = b.tablespace_name(+)
UNION ALL
SELECT h.tablespace_name AS tablespace_name,
       ROUND(SUM(h.bytes_free + h.bytes_used) / 1048576) megs_alloc,
       ROUND(SUM((h.bytes_free + h.bytes_used) - NVL(p.bytes_used, 0)) / 1048576) megs_free,
       ROUND(SUM(NVL(p.bytes_used, 0)) / 1048576) megs_used,
       ROUND((SUM((h.bytes_free + h.bytes_used) - NVL(p.bytes_used, 0)) / 
              SUM(h.bytes_used + h.bytes_free)) * 100) Pct_Free, 
       100 - ROUND((SUM((h.bytes_free + h.bytes_used) - NVL(p.bytes_used, 0)) / 
                    SUM(h.bytes_used + h.bytes_free)) * 100) pct_used,
       ROUND(SUM(f.maxbytes) / 1048576) max
FROM sys.v_$TEMP_SPACE_HEADER h, 
     sys.v_$Temp_extent_pool p,
     dba_temp_files f
WHERE p.file_id(+) = h.file_id
  AND p.tablespace_name(+) = h.tablespace_name
  AND f.file_id = h.file_id
  AND f.tablespace_name = h.tablespace_name
GROUP BY h.tablespace_name
ORDER BY 1;
```

**Output**

This query returns one row per tablespace showing allocation, usage, and capacity.

**Notes**

* Permanent tablespaces are retrieved from `dba_data_files` and `dba_free_space`.
* Temporary tablespaces are retrieved from dynamic performance views (`v_$TEMP_SPACE_HEADER`, `v_$Temp_extent_pool`).
* The `UNION ALL` combines both types for a complete picture.
* `% Free` approaching zero indicates potential space issues.

## Discovery Queries

### Find tables matching a name pattern

Search for all tables matching a pattern across all owners:

```sql
SET PAGESIZE 10000
COL OWNER FORMAT A15
COL TABLE_NAME FORMAT A30
COL TABLESPACE_NAME FORMAT A20
COL STATUS FORMAT A10

SELECT owner, table_name, tablespace_name, status
FROM all_tables
WHERE table_name LIKE 'PNR%'
ORDER BY owner, table_name;
```

**Parameters**

* `LIKE 'PNR%'` — adjust pattern as needed (% = wildcard)

**Output**

Lists all tables starting with "PNR" sorted by owner and table name.
