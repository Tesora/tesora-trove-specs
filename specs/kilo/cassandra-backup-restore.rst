..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

 Sections of this template were taken directly from the Nova spec
 template at:
 https://github.com/openstack/nova-specs/blob/master/specs/template.rst

==========================
Cassandra Backup & Restore
==========================

Launchpad blueprint:

https://blueprints.launchpad.net/trove/+spec/cassandra-backup-restore

Problem Description
===================

The Cassandra datastore currently does not support any backup/restore strategy.

Proposed Change
===============

The patch set will implement full backup/restore using the Nodetool [1]_
utility for Cassandra 2.1 [3]_.

Configuration
-------------

Following Cassandra configuration options will be updated:
   - backup/restore namespaces
   - backup_strategy

Database
--------

None

Public API
----------

None

Public API Security
-------------------

None

Internal API
------------

None

Guest Agent
-----------

We are implementing full backup using node snapshots following the procedure
outlined in the Nodetool manual [2]_.
Nodetool can take a snapshot of one or more keyspace(s).
A snapshot can be restored by moving all *\*.db* files from a snapshot
directory to the respective keyspace overwriting any existing files.

Backups are streamed to and from a remote storage as (TAR) archives.
We now outline the general procedure for creating and restoring
such an archive.

The Backup Procedure (creating a backup container):

1. Clear existing snapshots.

2. Take a snapshot of all keyspaces.

3. Collect all *\*.db* files from the snapshot directories package them into a
   single TAR archive.

   Transform the paths such that the backup can be restored simply by
   extracting the archive right to an existing data directory.
   This is to ensure we can always restore an old backup
   even if the standard guest agent data directory changes.

4. Compress and/or encrypt the archive as required.

The Restore Procedure (restoring data from a backup archive):

1. Create a new data directory.

2. Unpack the backup to that directory.

3. Update ownership of the restored files to the Cassandra user.

Additional Considerations:

When a backup is restored it carries the original cluster name with it
(instance ID for single node instances).
We have to update the configuration file to use the old name in order to
ensure that the restored node data still belongs to the original cluster
(this is also required by Cassandra - it would not start otherwise).

The 'superuser' *("root")* password stored in the system keyspace
needs to be reset before we can start up with restored data.

A general password reset procedure is:

1. Disable user authentication and remote access.

2. Restart the service.

3. Update the password in the 'system_auth.credentials' table.

4. Re-enable authentication and make the host reachable.

5. Restart the service.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Petr Malik <pmalik@tesora.com>

Milestones
----------

Liberty

Work Items
----------

1. Implement functionality needed for resetting cluster name and superuser password.
2. Implement backup/restore API calls.

Upgrade Implications
====================

None

Dependencies
============

The patch set will be building on functionality implemented in blueprints:
cassandra-database-user-functions and cassandra-configuration-groups

Testing
=======

Unittests will be added to validate implemented functions and non-trivial
codepaths.

Documentation Impact
====================

The datastore documentation should be updated to reflect the enabled features.
Also note the new configuration options - backup/restore namespaces and
backup_strategy for Cassandra datastore.

References
==========

.. [1] Documentation on Nodetool utility for Cassandra 2.1: http://docs.datastax.com/en/cassandra/2.1/cassandra/tools/toolsNodetool_r.html
.. [2] Manual on Backup and Restore for Cassandra 2.1: http://docs.datastax.com/en/cassandra/2.1/cassandra/operations/ops_backup_restore_c.html
.. [3] Documentation on Cassandra 2.1: http://docs.datastax.com/en/cassandra/2.1/cassandra/gettingStartedCassandraIntro.html