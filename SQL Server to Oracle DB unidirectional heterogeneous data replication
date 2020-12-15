# Configuring Heterogeneous Data Replication Between SQL Server and Oracle Database using GoldenGate

### Peter Song - Cloud Engineer


## Introduction

This post goes over how to set up a data replication pipeline using Oracle GoldenGate between a source SQL Server database and a target Oracle 19c database. Goldengate is a software pacakge for real-time data integration and replication, and is used for solutions that require high availability, real-time data integration, transactional change data capture, data replication, transformations, and verification between operational and analytical enterprise systems. 

#### Before You Begin

###### Environment details

- The SQL Server database is on a different machine than the GoldenGate instance. The SQL Server database will be accessed via Microsoft Remote Desktop
- The Windows instance was spun up through Oracle Cloud Infrastructure. If you want to replicate this environment you'll need access to a cloud tenancy.
- To allow remote connections you must add an ingress rule to your VCN to allow RDP connections
- The GoldenGate for SQL Server Marketplace Image is used. Specific version is Oracle_Golden_gate_Classic_19.1.0.0.200714_for SQLServer_on_Linux_OGGCORE
- The GoldenGate for Oracle DB is just the classic marketplace image: Oracle GoldenGate Classic Edition for Oracle 19.1.0.0.201013
- SQL Server version 2019 is used.

## Part 1: Configuring SQL Server Side

1. Connect to your SQL Server Instance and allow SQL Server and Windows Authentication Mode. (Right click the Instance on the left hand pop up window) Properties -> Security->check SQL Server and Windows Authentication mode.
   ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture1.png)

2. Create the Source Database

   ```
   create database testdb
   ```

3. Create a Login for the Source Database.

   ```sh
   CREATE LOGIN oggsourceuser with PASSWORD = N'password';
   ALTER SERVER ROLE sysadmin add oggsourceuser;
   USE testdb;
   CREATE SCHEMA oggsourceschema;
   ```

4. Set up ODBC Connection to your Windows Instance

   1. start menu -> Windows administrative tools ->ODBC Data Sources (64 Bit)
   2. Select System DSN
   3. Click Add
   4. Select ODBC Driver 17 for SQL Server
   5. Give it a name and select your server to connect to
   6. Select the third option (with SQL Server Authentication using a login ID and password entered by the user)
   7. Enter the login ID and password. Choose the database you want to conenct to. Accept the defaults and test Data Source. Hit OK to add the DSN
      ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture2.png)

5. Now you have to allow TCP/IP connection on your SQL Server Instance.

   1. Go to SQL Server Configuration Manager
   2. Go to SQL Server Network Configuration
   3. Select Protocols for MSSQLSERVER (or your named instance)
   4. Enable TCP/IP
      ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture4.png)
      5.Click TCP/IP and make sure TCP port 1433 is listed 
      ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture3.png)

6. Next, you must allow TCP connections to Port 1433 on your Windows Machine

   1. Control Panel -> System and Security -> Windows Defender Firewall -> Advanced Settings-> Inbound Rules ->New Rule ->Port-> TCP, Specific local ports (1433)-> Allow the connection-> Next-> give it a name.
   2. This is what it should look like when you try to add a rule. Port should be selected.
      ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture5.png)

7. Now that the port is open, you need to allow the app through the firewall

   1. Go to Control Panel -> System and Security ->Windows Defender Firewall -> Allow an App or Feature through Windows Defender Firewall -> Change Settings-> Allow another app-> Browse->Program Files-> Microsoft SQL Server-> your server name-> MSSQL->BINN->Select sqlservr
      ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Screen%20Shot%202020-12-09%20at%201.34.38%20PM.png)
      ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Screen%20Shot%202020-12-09%20at%201.34.50%20PM.png)

8. On your Machine where GoldenGate for SQL Server is installed you need to install Microsoft ODBC Drivers for linux. you can access the full documenatation on installation here: https://docs.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server?view=sql-server-ver15 .

   1. Edit the file /etc/passwd, to grant temporary shell access to the root user.

   ```
   $ -bash-4.2$ sudo vi /etc/passwd
   ```

   2. In the file /etc/passwd, change the value for the root user from /usr/sbin/nologin to /bin/bash. Save an close the file
   3. Using Microsoft's RedHat Enterprise Server Installation Instructions for adding the ODBC Drivers for Linux, perform the following steps with default values by answering 'Y' when prompted

   ```
   $ -bash-4.2$ sudo su
   $ -bash-4.2$ #Red Hat Enterprise Server 7
   $ -bash-4.2$ curl https://packages.microsoft.com/config/rhel/7/prod.repo > /etc/yum.repos.d/mssql-release.repo
   $ -bash-4.2$ exit
   $ -bash-4.2$ sudo yum remove unixODBC-utf16 unixODBC-utf16-devel #to avoid conflicts
   $ -bash-4.2$ sudo  ACCEPT_EULA=Y yum install msodbcsql17
   $ -bash-4.2$ sudo ACCEPT_EULA=Y yum install mssql-tools
   $ -bash-4.2$ echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bash_profile
   $ -bash-4.2$ source ~/.bashrc
   ```

9. Now it's time to configure an ODBC Connection (on the machine that hosts your goldengate SQL Server Instance)

   1. Create a template file for your data source

   ```
   -bash-4.2$ vi odbc_template_file.ini
   ```

   2. Describe the data source in the template file. In this example, ggtest03 is used as the DSN Name with DBLOGIN and SOURCEDB to connect to the database

   ```
   [ggtest03]
   Driver = ODBC Driver 17 for SQL Server
   Server = myserver,1433
   Database = dbname
   User = oggsourceuser
   Password = password
   ```

   3. Install the data source using the command

   ```
   -bash-4.2$ odbcinst -i -s -f odbc_template_file.ini
   ```

10. Creating GLOBALS File and Starting GGSCI

    1. Create a GLOBALS file in the OGG installation directory

    ```
    -bash-4.2$ cd ~/mssql
    -bash-4.2$ vi GLOBALS
    ```

    Add the following (minimum required) parameter to the GLOBALS file, substituting the correct value for the schema that you created earlier in the database. 

    ```
    GGSCHEMA oggsourceschema
    ```

    Save and close the GLOBALS file

    2. To start GGSCI, use the following command

    ```
    -bash-4.2$ ./ggsci
    ```

11. Enabling Supplemental Logging for a Source SQL Server

    1. From GGSCI connect to the source database with a SQL Server login with sysadmin privileges
       Example

    ```
    GGSCI> DBLOGIN sourcedb ggtest03 USERID oggsourceuser PASSWORD password
    GGSCI> ADD TRANDATA dbo.table1
    ```

    2. Using SQL server Management Studio, drop the SQL Server CDC Cleanup job by running the following against the database

    ```
    exec sys.sp_cdc_drop_job N 'cleanup';
    ```

    3. Execute the shell script to install the Oracle GoldenGate CDC cleanup tasks
       Syntax:

    ```
    ogg_cdc_cleanup.sh createJob userid password dbname server,port ggschema 
    ```

    Example

    ```
    GGSCI> shell ./ogg_cdc_cleanup_setup.sh createJob oggsourceuser password source_dbname myserver,1433 oggsourceschema
    ```

12. Configuring the Extract for SQL Server: The extract will capture transactional data from a source SQL server database and write it to a trail file. The extract only captures committed changes.

    1. Create an extract parameter file

    ```
    GGSCI> EDIT PARAMS extsql
    ```

    Sample:

    ```
    EXTRACT extsql
    SOURCEDB ggtest03 USERID oggsourceuser PASSWORD password
    EXTTRAIL ./dirdat/et
    TABLE dbo.table1;
    ```

    2. Add the Extract and Extract's local trail

    ```
    GGSCI> ADD EXTRACT extsql, TRANLOG, BEGIN NOW
    GGSCI> ADD EXTTRAIL ./dirdat/et, EXTRACT extsql
    ```

    3. Verify you are able to connect to SQL server and start the Extract

    ```
    GGSCI> DBLOGIN SOURCEDB ggtest03 USERID oggsourceuser PASSWORD password
    GGSCI (ogg19cmssql as oggsourceuser@GGTEST03) 2> START EXTRACT extsql
    GGSCI (ogg19cmssql as oggsourceuser@GGTEST03) 3> info all
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture6.png)

13. Configuring GoldenGate Pump Extract: It is HIGHLY recommendeed that you set up a data pump extract. The data pump acts as a secondary extract process where it reads the records from local trail files and delivers to the target system trail files through the collector. This way, if the primary extract fails, the pump extract will still be running

    1. Create the data pump extract

    ```
    GGSCI> edit param pmpsql
    
    EXTRACT PMPSSQL
    RMTHOST <target IP>, MGRPORT <Target Port>
    RMTTRAIL /home/opc/oracle19/dirdat/aa
    PASSTHRU
    TABLE dbo.table1
    ```

    2. Create the data pump group and remote extract trail file and start the data pump process

    ```
    GGSCI (ogg19cmssql as oggsourceuser@GGTEST03) 1> Add extract pmpsql exttrailsource ./dirdat/et
    GGSCI (ogg19cmssql as oggsourceuser@GGTEST03) 2> Add rmttrail /home/opc/oracle19/dirdat/aa, extract pmpsql
    GGSCI (ogg19cmssql as oggsourceuser@GGTEST03) 3> Start extract pmpsql
    ```

    3. Check that both the extract and data pump are running:

    ```
    GGSCI (ogg19cmssql as oggsourceuser@GGTEST03) 4> info all
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture7.png)

## Part 2: Configuring Oracle DB Side

14. First we need to enable supplemental logging in our Oracle Database. Connect to your Oracle DB on SQL Developer

    1. Lets see if archive logging is enabled in the database

    ```
    archive log list;
    ```

    If it says enabled, then the next step can be skipped

    2. If disabled, run:

    ```
    alter database archivelog;
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/archivelogScreen%20Shot%202020-12-09%20at%202.24.34%20PM.png)

    3. Next we need to check if goldengate replication is turned on in the database.

    ```
    show parameter enable_goldengate_replication
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture9.png)
    If the value is set to true, the command below can be skipped

    ```
    alter system set enable_goldengate_replication=TRUE scope=both;
    ```

    4. If the above steps were done correctly the query below should return...

    ```
    select name, cdb, supplemental_log_data_all,force_logging from v$database; 
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture10.png)

    5. Now in case you weren't in cdb$root, switch to cdb$root with

    ```
    alter session set container = cdb$root;
    ```

    6. We need to add supplemental log data 

    ```
    Alter database add supplemental log data (all) columns;
    Alter database force logging;
    Alter database switch logfile;
    ```

15. Now we need to create our goldengate users. Goldengate requires a dedicated user for GoldenGate processes (storing checkpoint tables, etc). The user must be a common user. 

    ```
    create user C##GGADMIN identified by password default tablespace system temporary tablespace temp container =ALL;
    grant dba to C##GGADMIN container = all;
    exec dbms_goldengate_auth.grant_admin_privilege(grantee=> 'C##GGADMIN', container => 'all');
    ```

16. NOTE: The target tables must be in a pdb and NOT a CDB, so pick a pdb and alter the session to that pdb.

    ```
    show pdbs;
    ```

    pick a pdb

    ```
    alter session set container = targetdb_pdb1;
    ```

    create a user within the PDB

    ```
    create user user1 identified by password default tablespace system temporary tablespace temp container = current;
    grant connect,resource to user1;
    grant create session to user1;
    grant select any dictionary to user1;
    grant create view to user1;
    grant execute on dbms_lock to user1;
    alter user peter quota unlimited on users;
    grant unlimited tablespace to user1;
    ```

17. Now recreate the table from the SQL Server exactly, within the PDB and for the user we just created
    Example:

    ```
    create table user1.Inventory(
        product_id INT PRIMARY KEY,
        product_name VARCHAR2(100) NULL,
        price NUMBER(18,2) NULL,
        quantity INT NULL
    );
    ```

18. Now we have to connect GoldenGate to our Oracle Database

    1. Let's set up our tnsnames.ora file

    ```
    -bash-4.2$ cd /u01/app/client/oracle19/network/admin
    ```

    2. If /network/admin doesn't exist, create those directories.
    3. Then within the admin folder, create the tnsnames.ora file

    ```
    -bash-4.2$ vi tnsnames.ora
    DB19C_PDB=(DESCRIPTION=(CONNECT_TIMEOUT=5)(TRANSPORT_CONNECT_TIMEOUT=3)(RETRY_COUNT=3)(ADDRESS_LIST=(LOAD_BALANCE=on)(ADDRESS=(PROTOCOL=TCP)(HOST=IP_ADDRESS)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=targetDB_pdb1.xxxxxxxxxx.vcnxxxxxxxxx.oraclevcn.com)))
    ```

    4. Let's save our changes

    ```
    -bash-4.2$ source ~/.bash_profile
    ```

19. Now let's add a credentialstore so we can login quickly to the database using a useridalias

    ```
    GGSCI> add credentialstore
    GGSCI> alter credentialstore add user userid@tns_connect_identifier  PASSWORD password alias any_alias_name
    GGSCI> info credentialstore
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture11.png)

20. Next we need to make sure that the target is reachable from the source. First, let's see which port is being used by the GoldenGate manager (target side) 

    1. 

    ```
    GGSCI (ogg19cora) 1> view param mgr
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture12.png)

    2. Here I am using port 7810 for the manager and opened up 7811-7830 as dynamic ports. Depending on your configuration, these ports may be blocked by your firewall. To check which ports are open

    ```
    -bash-4.2$ sudo firewall-cmd --list-ports
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture13.png)
    If you don’t see the appropriate ports open, you need to manually open them. If everything looks good, skip the next step

    3. To open the port you want

    ```
    -bash-4.2$ sudo firewall-cmd --permanent --add-port=<port_number>/tcp
    -bash-4.2$ sudo firewall-cmd --reload
    ```

21. Now that the appropriate ports are open, and the bash_profile is up to dat, we can finally get to configuring our replicat! The replicat is what reads the data from the trail fies.

    1. Login to the Oracle DB using the useridalias
       ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture14.png)
    2. Next, let's add a checkpointtable to our c##ggadmin user

    ```
    GGSCI (ogg19cora as c##ggadmin@targetDB/TARGETDB_PDB1) 1> add checkpointtable c##ggadmin.chkpt
    ```

    3. Now let's create the replicat parameter file

    ```
    REPLICAT rep1
    useridalias ggadmin
    ASSUMETARGETDEFS
    MAP dbo.table1, TARGET targetdb_pdb1.user1.Inventory;
    ```

    4. Time to create and start the replicat process

    ```
    GGSCI (ogg19cora as c##ggadmin@targetDB/TARGETDB_PDB1) 1> add replicat rep1/home/opc/oracle19/dirdat/aa checkpointtable c##ggadmin.chkpt
    GGSCI (ogg19cora as c##ggadmin@targetDB/TARGETDB_PDB1) 2> start replicat rep1
    GGSCI (ogg19cora as c##ggadmin@targetDB/TARGETDB_PDB1) 3> info rep1
    ```

    ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture15.png)

22. Now for the moment of truth. If everything was configured correctly, you should be able to go into your source SQL server database and make an insertion/make a change and it should show up on your target Oracle DB. 

    1. Insert something into your source DB and run stats rep1 to see if it captured changes. After 1 insert it should look like this screenshot below: 
       ![](https://objectstorage.us-ashburn-1.oraclecloud.com/n/orasenatdoracledigital01/b/bucket-20201209-1145/o/Picture16.png)
    2. If you check your Oracle DB and that data is there, you’ve successfully configured a heterogeneous replication from a SQL Server DB to an Oracle DB!
