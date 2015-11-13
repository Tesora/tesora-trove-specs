..
    This work is licensed under a Creative Commons Attribution 3.0 Unported
    License.

    http://creativecommons.org/licenses/by/3.0/legalcode

..


=========================================
Configuration groups for Oracle datastore
=========================================

Configuration groups are a powerful tool for customizing  databases managed by
Trove. This spec describes an implementation to support configuration groups
for the Oracle datastore.

Launchpad Blueprint:
https://blueprints.launchpad.net/trove/+spec/oracle-config-groups


Problem Description
===================

Oracle Database provides "initialization parameters" for database system
management. To allow users to modify these the Oracle datastore in Trove must
support configuration groups. Attaching a configuration group should provide a
list of key-value pairs that map to these settings. Attaching a group at
instance-create should bring the database up with the settings. Attaching to a
running instance should attempt to apply the settings online. If one or more
settings are not modifiable then the database should be restarted.

Proposed Change
===============

A text parameter file (PFILE) will be maintained by Trove, and a server
parameter file (SPFILE) will be generated as needed. Any configuration group
actions will cause the PFILE to be updated. If all the affected parameters
are modifiable then an ALTER SYSTEM command will be issued which updates the
SPFILE automatically. If the a non-modifiable parameter is changed then the
database will be shut down, a new SPFILE will be generated, and the database
will be restarted.

Not all configuration parameters will be supported. The list will be a
exclude any file path, cluster, and some other parameters. Many of the
settings do not make sense in the Trove guest environment - such as moving
data and backup locations, or managing resources for a shared environment.

Configuration
-------------

N/A

Database
--------

The configuration groups will be stored in the Trove database using the
existing configuration groups models.

Public API
----------

The configuration groups will be managed via the existing APIs.

Public API Security
-------------------

Validation of parameters and values will be done as needed.

Python API
----------

N/A

CLI (python-troveclient)
------------------------

N/A

Internal API
------------

The Oracle datastore will now process configuration group requests.

Guest Agent
-----------

The configuration group key/value pairs will be stored in a PFILE managed by
the configuration manager. This will handle editing and versioning. When a
group is added an SPFILE will be generated to test the new parameters.
ALTER SYSTEM, stop and start the database will be handled by the guestagent.

Alternatives
------------

N/A

Dashboard Impact (UX)
=====================

No additional work is needed for the Dashboard. The current configurations
interface is sufficient.

Implementation
==============

Assignee(s)
-----------

Matthew Van Dijk

Milestones
----------

N/A

Work Items
----------

A single commit will implement the changes.

Upgrade Implications
====================

N/A

Dependencies
============

N/A

Testing
=======

Appropriate unit and integration tests will be added.

Documentation Impact
====================

The list of managed parameters should be documented.

References
==========

.. [1] `11g parameters <https://docs.oracle.com/cloud/latest/db121/REFRN/initparams.htm#g1195420>`_
.. [2] `12c parameters <https://docs.oracle.com/database/121/REFRN/GUID-FD266F6F-D047-4EBB-8D96-B51B1DCA2D61.htm#REFRN-GUID-FD266F6F-D047-4EBB-8D96-B51B1DCA2D61>`_

Appendix
========

