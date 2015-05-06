..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

 Sections of this template were taken directly from the Nova spec
 template at:
 https://github.com/openstack/nova-specs/blob/master/specs/template.rst

==============================
Cassandra Configuration Groups
==============================

Launchpad blueprint:

https://blueprints.launchpad.net/trove/+spec/cassandra-configuration-groups

Problem Description
===================

The Cassandra datastore currently does not support configuration groups.

Proposed Change
===============

The patch set will implement configuration groups for Cassandra 2.1.

Configuration
-------------

None

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

Cassandra stores its configuration in 'cassandra.yaml' file
(commonly in '/etc/cassandra').
The node (datastore service) has to be restarted for any changes to the
configuration file to take effect. All configuration changes will therefore be
requiring database restart and 'apply_overrides' will be implemented as no-op.

Overrides will be implemented by replacing the current file with an
updated one.
The old file will be backed up in the same directory (as *\*.old*) and
restored on configuration reset.

We will make effort to enable most configuration properties except options that
require functionality currently not supported or may, when misconfigured,
easily render the instance inaccessible by Trove.

The user should be able to specify configurations properties as standard Python
YAML objects - key-value pairs and dicts.

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

1. Implement functionality to handle (read/write/update) YAML files.
2. Implement configuration-related manager API calls.

Upgrade Implications
====================

None

Dependencies
============

The patch set will be building on functionality implemented in blueprint:
cassandra-database-user-functions

Testing
=======

Unittests will be added to validate implemented functions and non-trivial
codepaths.

Documentation Impact
====================

The datastore documentation should be updated to reflect the enabled features.

References
==========

.. [1] Documentation on Cassandra 2.1: http://docs.datastax.com/en/cassandra/2.1/cassandra/gettingStartedCassandraIntro.html
.. [2] Documentation on Cassandra 2.1 configuration properties: http://docs.datastax.com/en/cassandra/2.1/cassandra/configuration/configTOC.html