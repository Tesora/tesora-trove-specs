..
    This work is licensed under a Creative Commons Attribution 3.0 Unported
    License.

    http://creativecommons.org/licenses/by/3.0/legalcode


============================
Oracle Backup and Restore
============================

This blueprint documents the changes needed in Trove to support backup and
restore of a single Oracle 11g datastore instance.


Problem Description
===================

Being able to back up and restore databases is a fundamental requirement for
any DBMS. Trove currently does not support such functionality for the Oracle
datastore and it needs to be implemented.


Proposed Change
===============

Create new trove guestagent modules to issue Oracle RMAN (Recovery Manager)
BACKUP commands against all databases within the instance. Hot backups will
be performed, meaning all databases will remain online while backup is
performed. Backup sets will include the data (dbf) files, control file, SP
file, and password file.

This blueprint will implement the following features for the Oracle datastore:

**Level 0 Incremental Backup (Full Backup):**

This backup method will be initiated by the trove backup-create command. It
will perform a complete backup of all databases within the instance.

**Level 1 Incremental Differential Backup (Incremental Backup):**

This backup method will be initiated by the trove backup-create command with
the --parent <parent-backup-id> parameter. It will back up all databases within
the trove instance. However, this option will only back up the blocks changed
since the level 0 or level 1 backup with <parent-backup-id>. Checks will be
implemented to only allow referencing a --parent backup that does not have a
child.

**Database Restore:**

Restoration is initiated by the trove create command with the
--backup <backup-id> parameter. It will restore all databases in the backup.

**The following trove commands will also support the Oracle datastore.
No changes are needed:**

* backup-delete
* backup-list
* backup-show
* backup-list-instance

*Sample Use Case:*

Let's say we have two databases, the following is the list of backup files:

$DEFAULT_BACKUP_DIR/TESTDB15/backupset/2015_07_07

* o1\_mf\_ncsn0\_LEVEL0\_SUN\_bsr72186\_.bkp
* o1\_mf\_nnnd0\_LEVEL0\_SUN\_bsr71q4p\_.bkp

$DEFAULT_BACKUP_DIR/TESTDB16/backupset/2015_07_07

* o1\_mf\_ncsn0\_LEVEL0\_SUN\_bsr64g62\_.bkp
* o1\_mf\_nnnd0\_LEVEL0\_SUN\_bsr6462c\_.bkp

$ORACLE_HOME/dbs

* orapwtestdb16 (the password file)
* spfiletestdb16.ora (the SP file)

An encrypted tar file that contains the above files will be created. This tar
file will be stored in Swift.

On restore, the tar file will be retrieved from Swift, and the backup files
archived within will be restored at their original location by RMAN. The
password and SP files will be restored to the $ORACLE_HOME/dbs directory by the
standard file system copy command.

Unlike MySQL, there's no need to store the log sequence number (lsn) along with
the Swift object in the metadata. The latest control file has all the backup
details, RMAN will be able to perform automatic restores as long as the backup
files are in tact at the expected location.

Entries for the restored databases will also be added back to the index file
(/etc/oratab). The database will also be registered back with the listener.

Finally, trove create currently does not put new databases into archivelog
mode, which is a requirement of performing RMAN hot backups. Changes will be
made to enable archivelog mode on database creation.

Configuration
-------------

The default values for backup_strategy, backup_namespace and restore_namespace
for Oracle will change to point to the appropriate locations.

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

This blueprint enables the use of backup CLI commands on Oracle datastores.
However, no code change is needed on the python-troveclient.

Internal API
------------

None

Guest Agent
-----------

New Oracle backup and restore strategies will be added to the guestagent.

The guestagent will shell out and execute RMAN command in this format against
a database to perform backup and restores:
::

	rman target SYS/pswd@dbname <<EOF
	run {
	<RMAN_COMMANDS>
	}
	EXIT;
	EOF


Where <RMAN_COMMANDS> can be:

For level 0 (full) backups:
::

	BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL=0
	DATABASE TAG='LEVEL0-<TIMESTAMP>';

For level 1 (incremental) backups:
::

	BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL=1
	DATABASE TAG='LEVEL1-<TIMESTAMP>';

For database restores:
::

	SET DBID <DB_ID>
	STARTUP NOMOUNT;
	RESTORE CONTROLFILE FROM '<LATEST_CONTROL_FILE>';
	STARTUP MOUNT;
	CROSSCHECK BACKUP;
	RESTORE DATABASE;
	RECOVER DATABASE;
	ALTER DATABASE OPEN RESETLOGS;

The create_backup and _perform_restore methods in
trove/guestagent/datastore/oracle/manager.py
will be implemented.

The following modules will also be implemented:

* trove/guestagent/strategies/backup/oracle_impl.py
* trove/guestagent/strategies/restore/oracle_impl.py

Alternatives
------------

It is possible to perform backup and restore with SQL commands via sql*plus
and the cx_oracle Python library. However, incremental backups can only be
performed with RMAN, leaving RMAN the only solution for implementing this
blueprint.

There is also no Oracle tools which supports streaming backup data directly
to Swift. RMAN is only capable of streaming backup data to tape devices.
Leaving a two stage backup (i.e. backup to a temp location then upload to
Swift) the best option available. We'll instruct RMAN to compress the data to
minimize the size of the backup sets.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  schang

Milestones
----------

Target Milestone for completion:
  EE-1.5

Work Items
----------

* Implement new modules and methods
* Implement test cases


Upgrade Implications
====================

Backup requires Oracle database instances running in ARCHIVELOG mode. It will
not work for existing Oracle instances because they are not running in
ARCHIVELOG mode.


Dependencies
============

The backup/restore solution proposed by this blueprint depends on RMAN, which
comes with Oracle pre-installed.


Testing
=======

Test cases used for testing backup/restore for MySQL will be adapted to run
against Oracle.


Documentation Impact
====================

The documentation should reflect that Oracle backup/restore is supported for
single instances.


References
==========

.. [1] http://www.thegeekstuff.com/2013/08/oracle-rman-backup/

.. [2] http://www.thegeekstuff.com/2014/11/oracle-rman-restore/

.. [3] https://www.youtube.com/watch?v=dJVKrswatV0

.. [4] https://www.youtube.com/watch?v=w_cT4mdTf_U

.. [5] https://community.oracle.com/thread/1116590

.. [6] http://www.online-database.eu/index.php/recovery-manager-rman/83-clone-database-with-rman-on-another-host-option-3

.. [7] http://docs.oracle.com/cd/E11882_01/server.112/e25789/intro.htm#CNCPT914


