..
    This work is licensed under a Creative Commons Attribution 3.0 Unported
    License.

    http://creativecommons.org/licenses/by/3.0/legalcode


====================
Oracle Replication
====================

This blueprint documents the changes needed in Trove to support replication of
Oracle 11g and 12c datastore instances.


Problem Description
===================

Trove can only create single Oracle instances at the moment. Although
backup/restore is already available, being able to replicate data from a
master to multiple slaves and perform quick switch/fail over is essential for
disaster recovery. This functionality will be addressed in this blueprint.


Proposed Change
===============

Create new trove guestagent modules that configure Oracle databases to work
with Data Guard replication. RMAN active duplicate will be used to clone
the master database into slaves, no backup will be needed during the clone
process.

This blueprint will implement the following features for the Oracle datastore:

*Note:*
The following node naming conventions will be used throughout this blueprint:

* DB11G - Master Node
* DB11G_STBY - Slave Node #1
* DB11G_STBY2 - Slave Node #2


**Create Slave Node(s)**

*Trove command:*
::

	trove create <instance-name> <flavor-id> --size=<volume-size>
	--datastore_version <datastore-version> --datastore <datastore-name>
	--replica_of <master-instance-id> --replica_count <number-of-slaves-to-create>

*Procedure:*

Enable archivelog mode and force logging on Master:

::

	SHUTDOWN IMMEDIATE;
	STARTUP MOUNT;
	ALTER DATABASE ARCHIVELOG;
	ALTER DATABASE OPEN;
	ALTER DATABASE FORCE LOGGING;


Enable archive log shipping on Master by setting the proper system parameters:
::

	ALTER SYSTEM SET LOG_ARCHIVE_CONFIG=
		'DG_CONFIG=(DB11G,DB11G_STBY,DB11G_STBY2)';
	ALTER SYSTEM SET LOG_ARCHIVE_DEST_2=
		'SERVICE=db11g_stby NOAFFIRM ASYNC
		VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
		DB_UNIQUE_NAME=DB11G_STBY';
	ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2=ENABLE;
	ALTER SYSTEM SET LOG_ARCHIVE_DEST_3='SERVICE=db11g_stby2 NOAFFIRM ASYNC
		VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
		DB_UNIQUE_NAME=DB11G_STBY2';
	ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_3=ENABLE;
	ALTER SYSTEM SET LOG_ARCHIVE_FORMAT='%t_%s_%r.arc' SCOPE=SPFILE;
	ALTER SYSTEM SET LOG_ARCHIVE_MAX_PROCESSES=30;
	ALTER SYSTEM SET REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE SCOPE=SPFILE;
	ALTER SYSTEM SET FAL_SERVER='DB11G_STBY','DB11G_STBY2';
	ALTER SYSTEM SET DB_FILE_NAME_CONVERT=
		'DB11G_STBY','DB11G','DB11G_STBY2','DB11G' SCOPE=SPFILE;
	ALTER SYSTEM SET LOG_FILE_NAME_CONVERT=
		'DB11G_STBY','DB11G','DB11G_STBY2','DB11G'  SCOPE=SPFILE;
	ALTER SYSTEM SET STANDBY_FILE_MANAGEMENT=AUTO;


Create standby redo logs on Master:
::

	ALTER DATABASE ADD STANDBY LOGFILE
		('/u01/app/oracle/oradata/DB11G/standby_redo01.log') SIZE 50M;
	ALTER DATABASE ADD STANDBY LOGFILE
		('/u01/app/oracle/oradata/DB11G/standby_redo02.log') SIZE 50M;
	ALTER DATABASE ADD STANDBY LOGFILE
		('/u01/app/oracle/oradata/DB11G/standby_redo03.log') SIZE 50M;
	ALTER DATABASE ADD STANDBY LOGFILE
		('/u01/app/oracle/oradata/DB11G/standby_redo04.log') SIZE 50M;


Create a tnsnames.ora file on the master node and all slave nodes. See
Appendix A for sample file content.


Create a listener.ora file similar to this on the master node and all
slave nodes. See Appendix B for sample file content.


Create a PFILE out of the master's spfile for each slave:
::

	CREATE PFILE='/tmp/initDB11G_stby.ora' FROM SPFILE;


Amend the PFILE making the entries relevant to each slave::


	*.db_unique_name='DB11G_STBY'
	*.fal_server='DB11G','DB11G_STBY2'
	*.log_archive_dest_2='SERVICE=db11g ASYNC
		VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=DB11G'
	*.log_archive_dest_3='SERVICE=db11g_STBY2 ASYNC
		VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
		DB_UNIQUE_NAME=DB11G_STBY2'


Copy the appropriate PFILE onto each slave with scp.


Create these directories on each slave:

* /u01/app/oracle/oradata/DB11G
* /u01/app/oracle/fast_recovery_area/DB11G
* /u01/app/oracle/admin/DB11G/adump


Clone the master's control file with this command:
::

	ALTER DATABASE CREATE STANDBY CONTROLFILE AS '/tmp/db11g_stby.ctl';


Copy and place each control file onto the dbs and fast_recovery_area
directory of each slave node, rename the file accordingly.


Copy the password file from master to all slaves.


Start the listener on each slave:
::

	lsnrctl start


Copy data from Master to Slave with RMAN:
::

	rman TARGET sys/password@/<IP of PRIMARY NODE>/DB11G AUXILIARY
		sys/password@DB11G_STBY

	DUPLICATE TARGET DATABASE
	  FOR STANDBY
	  FROM ACTIVE DATABASE
	  DORECOVER
	  SPFILE
	    SET db_unique_name='DB11G_STBY' COMMENT 'Is standby'
	    SET LOG_ARCHIVE_DEST_2='SERVICE=db11g ASYNC
		VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=DB11G'
	    SET LOG_ARCHIVE_DEST_3='SERVICE=db11g_STBY2 ASYNC
		VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
		DB_UNIQUE_NAME=DB11G_STBY2'
	    SET FAL_SERVER='DB11G','DB11G_STBY2' COMMENT 'Is primary'
	  NOFILENAMECHECK;


Start the log apply process on Slave:
::

	ALTER DATABASE OPEN READ ONLY;
	ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE
		DISCONNECT FROM SESSION;


**Promote Slave to Master (Switchover)**

*Trove command:*
::

	trove promote-to-replica-source <instance-id>


*Procedure:*

Switch the Master to Slave by issuing the following sql*plus commands on the
Master:
::

	ALTER DATABASE COMMIT TO SWITCHOVER TO STANDBY;
	SHUTDOWN IMMEDIATE;
	STARTUP NOMOUNT;
	ALTER DATABASE MOUNT STANDBY DATABASE;
	ALTER DATABASE OPEN READ ONLY;
	ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
		DISCONNECT FROM SESSION;


Switch the Slave to Master by issuing the following sql*plus commands on
the Slave:
::

	ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY;
	SHUTDOWN IMMEDIATE;
	STARTUP;


**Eject Master (Failover)**

*Trove command:*
::

	trove eject-replica-source <instance-id>


*Procedure:*

On all slaves, set the LOG_ARCHIVE_DEST_STATE_n system parameter of the Master
node that needs to be ejected to 'DEFER'.


Select a running slave and issue this sql*plus command to promote it into
Master:
::

	ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH;
	ALTER DATABASE ACTIVATE STANDBY DATABASE;


Restart the Oracle service on all nodes.


**Remove Slave Node**

*Trove command:*
::

	trove detach-replica <instance-id>


*Procedure:*

Disable the log apply process on the slave to be detached.


Adjust all LOG_ARCHIVE_DEST_* system parameters to exclude the slave to
be detached.


Restart the Oracle service on all nodes.


**Admin user management**

Admin user privileges are being stored in password files located at
$ORACLE_HOME/dbs/orapw<db-name>. Any changes to admin privileges (e.g. sys
user password changes, root enabling, assigning admin access to a regular
user) would require the password files on all nodes in the replication pool
to be updated and synced.


Configuration
-------------

The default values for replication_strategy and replication_namespace for
Oracle will change to point to the appropriate locations.

Database
--------

None

Public API
----------

None

Public API Security
-------------------

None

Python API
----------

None

CLI (python-troveclient)
------------------------

This blueprint enables the use of replication CLI commands on Oracle
datastores. However, no code change is needed on the python-troveclient.

Internal API
------------

None

Guest Agent
-----------

New Oracle replication strategies will be added to the guestagent.

The guestagent will shell out and execute RMAN command in this format to
clone master database into slaves:
::

	rman TARGET sys/password@/<IP of MASTER>/<MASTER_DB> AUXILIARY
		sys/password@<SLAVE_DB> <<EOF
	run {
	<DUPLICATE TARGET DATABASE COMMANDS>
	}
	EXIT;
	EOF

Other Data Guard related sql*plus Data Guard commands will be executed via the
cx_oracle Python library.


Alternatives
------------

Oracle Data Guard Broker can be used to perform database switchover and
failover. Since it is meant to be tool that simplifies management of
replication DBA tasks, the Data Guard Broker does not offer the level of
control granularity needed by trove to programmatically manage replication
tasks. Data Guard Broker also requires configuration of custom "_DGMGRL"
listeners, and it is unclear how we can programmatically recover from failed
switchover operations that the Broker performed.  For the above reasons, it
has been determined Data Guard Broker is not a viable alternative to implement
this blueprint.


Dashboard Impact (UX)
=====================

Dashboard needs to be tested for Oracle replication support. Changes may
need to be implemented.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  schang

Milestones
----------

Target Milestone for completion:
  EE-1.7

Work Items
----------

* Implement new modules and methods
* Implement test cases


Upgrade Implications
====================

Data Guard replication requires Oracle database instances running in ARCHIVELOG
mode with FORCE LOGGING enabled. Existing instances created by EE-1.6 should
already be running in ARCHIVELOG mode.  FORCE LOGGING will be enabled when the
replication pool is being created.


Dependencies
============

The replication solution proposed by this blueprint depends on Oracle Active
Data Guard and RMAN, which comes pre-installed with Oracle Enterprise Edition
11g and 12c.


Testing
=======

Test cases used for testing replication for MySQL will be adapted to run
against Oracle.


Documentation Impact
====================

The documentation should reflect that Oracle replication is only supported for
single instances.


References
==========

.. [1] https://oracle-base.com/articles/11g/data-guard-setup-11gr2

.. [2] http://www.oracle.com/au/products/database/maa10gr2multiplestandbybp-1-131937.pdf

.. [3] http://www.amazon.com/Oracle-Guard-11gR2-Administration-Beginners/dp/1849687900

.. [4] http://www.oracle.com/webfolder/technetwork/tutorials/obe/db/11g/r2/prod/ha/dataguard/physstby/physstdby.htm

.. [5] http://www.oracle.com/webfolder/technetwork/tutorials/obe/db/11g/r2/prod/ha/dataguard/dg_broker/dg_broker.htm


Appendix
========

Appendix A - Sample tnsnames.ora file
-------------------------------------
::

	DB11G =
	  (DESCRIPTION =
	    (ADDRESS_LIST =
	      (ADDRESS = (PROTOCOL = TCP)(HOST = ol5-112-dga1)(PORT = 1521))
	    )
	    (CONNECT_DATA =
	      (SERVICE_NAME = DB11G.WORLD)
	    )
	  )

	DB11G_STBY =
	  (DESCRIPTION =
	    (ADDRESS_LIST =
	      (ADDRESS = (PROTOCOL = TCP)(HOST = ol5-112-dga2)(PORT = 1521))
	    )
	    (CONNECT_DATA =
	      (SERVICE_NAME = DB11G.WORLD)
	    )
	  )
	DB11G_STBY2 =
	  (DESCRIPTION =
	    (ADDRESS_LIST =
	      (ADDRESS = (PROTOCOL = TCP)(HOST = ol5-112-dga3)(PORT = 1521))
	    )
	    (CONNECT_DATA =
	      (SERVICE_NAME = DB11G.WORLD)
	    )
	  )


Appendix B - Sample listener.ora file
-------------------------------------
::

	SID_LIST_LISTENER =
	  (SID_LIST =
	    (SID_DESC =
	      (GLOBAL_DBNAME = DB11G.WORLD)
	      (ORACLE_HOME = /u01/app/oracle/product/11.2.0/db_1)
	      (SID_NAME = DB11G)
	    )
	  )

	LISTENER =
	  (DESCRIPTION_LIST =
	    (DESCRIPTION =
	      (ADDRESS = (PROTOCOL = TCP)(HOST = ol5-112-dga1.localdomain)
		(PORT = 1521))
	    )
	    (DESCRIPTION =
	      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
	    )
	  )

	ADR_BASE_LISTENER = /u01/app/oracle



