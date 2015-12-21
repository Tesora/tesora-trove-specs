..
    This work is licensed under a Creative Commons Attribution 3.0 Unported
    License.

    http://creativecommons.org/licenses/by/3.0/legalcode

    Sections of this template were taken directly from the Nova spec
    template at:
    https://github.com/openstack/nova-specs/blob/master/specs/template.rst

..
    This template should be in ReSTructured text. The filename in the git
    repository should match the launchpad URL, for example a URL of
    https://blueprints.launchpad.net/trove/+spec/awesome-thing should be named
    awesome-thing.rst.

    Please do not delete any of the sections in this template.  If you
    have nothing to say for a whole section, just write: None

    Note: This comment may be removed if desired, however the license notice
    above should remain.


====================
Replication Affinity
====================

.. If section numbers are desired, unindent this
    .. sectnum::

.. If a TOC is desired, unindent this
    .. contents::

Nova has the ability to control whether new instances are created on the same
hypervisor (affinity) or on different hypervisors (anti-affinity).  This
behaviour is useful when setting up replication networks, so it is proposed to
add support for it into Trove.

Launchpad Blueprint:
https://blueprints.launchpad.net/trove/+spec/replication-affinity


Problem Description
===================

At times it is desirable to control where replicas in a replication set are
created.  In the case of read slaves, there may be a requirement to ensure that
all replicas are on the same hypervisor, whereas for fault tolerance it may be
desired that all replicas are created on different hypervisors.  Replication
affinity seeks to address this issue.  (Granted, anti-affinity seems more
relevant - and desirable - however the two go hand-in-hand as will be seen in
the next section.)


Proposed Change
===============

As mentioned above, Nova has the capability of placing new instances on the
same hypervisor, or on different ones.  Within Nova, this is handled through
the use of 'server groups.' [1]_  This will require sending a server_group
'hint' to the nova_client.servers.create call.  (This will be similar to how
secgroups are created now.)

Trove will perform the following actions to accomplish this:

* The create_instance method in the Task Manager will be changed to include
  a 'locality' argument.

.. code-block:: python

    def create_instance(self, context, instance_id, name, flavor,
                        image_id, databases, users, datastore_manager,
                        packages, volume_size, backup_id, availability_zone,
                        root_password, nics, overrides, slave_of_id,
                        cluster_config, volume_type, locality):

* This argument will be used to create a server group with the
  corresponding policy.
* The server group details will be converted to a Nova 'hint.'
* This hint will be passed into the Nova client in the create call in
  taskmanager/models.py.

Note: If affinity is chosen and the hypervisor does not have enough resources,
then some of the replicas will fail to create.  The same goes for choosing
anti-affinity and there are not enough available hypervisors.  There may be
other cases where Nova is unable to create the replica (affinity chosen, but
different AZ's); in these cases the replica will likely fail as well.

In order to easier create a replication set with affinity (or anti-affinity),
the behaviour for create will also be changed to allow 'None' as the value for
--replica_of, if --replica_count is specified.  This will have the effect of
creating the 'master' node first before creating the replicas.  It will also
mean that taking a backup is unneccessary, speeding up the entire process.

The Trove show command will also be modified to show the 'locality' value (i.e.
the server_group policy), if it exists.

Configuration
-------------

No changes

Database
--------

No changes

Public API
----------

An attribute 'locality' will be added to the data payload sent during a
'create' command.  The request/response will look like:

Request::

    POST v1/{tenant_id}/instances
    {
        "instance":
        {
            "volume":
            {
                "type": null,
                "size": 1
            },
            "flavorRef": 7,
            "name": "myinst",
            "replica_count": 2,
            "locality": "affinity"
        }
    }

Response::

    {
        "instance":
        {
            "status": "BUILD",
            "updated": "2015-12-13T12:36:59",
            "name": "myinst",
            "links":
            [
                {
                    "href": "https://10.240.64.151:8779/v1.0/<tenant>/instances/<id>",
                    "rel": "self"
                },
                {
                    "href": "https://10.240.64.151:8779/instances/<id>",
                    "rel": "bookmark"
                }
            ],
            "created": "2015-12-13T12:36:59",
            "id": "<id>",
            "volume":
            {
                "size": 1
            },
            "flavor":
            {
                "id": "7",
                "links":
                [
                    {
                        "href": "https://10.240.64.151:8779/v1.0/<tenant>/flavors/7",
                        "rel": "self"
                    },
                    {
                        "href": "https://10.240.64.151:8779/flavors/7",
                        "rel": "bookmark"
                    }
                ]
            },
            "datastore":
            {
                "version": "5.6",
                "type": "mysql"
            },
            "locality": "affinity"
        }
    }

The 'show' command will also now return an attribute 'locality' that will
look like the one returned from the create command.  Although this doesn't
represent a 'change' in the API, it is listed here for reference.  Since the
flag will only be returned if the locality has been specified, it will be fully
backwards compatible (as older instances will not have locality associated with
them, they would not return the flag regardless).
Note: An alternate name of the flag could be 'policy' instead of 'locality.'

Public API Security
-------------------

No impact

Python API
----------

A new argument 'locality' will be added to the Trove create command (this will
be passed through to the Nova client as a hint by the Task Manager, after
creating the corresponding server group).  The new Python API signature will
be:

.. code-block:: python

    def create(self, name, flavor_id, volume=None, databases=None, users=None,
               restorePoint=None, availability_zone=None, datastore=None,
               datastore_version=None, nics=None, configuration=None,
               replica_of=None, slave_of=None, replica_count=None,
               locality=None):

CLI (python-troveclient)
------------------------

The create command will now accept a --locality flag that can be one of two
values: affinity and anti-affinity.  The command would look like:

.. code-block:: bash

    trove create my_instance 7 --size 1 --locality affinity

Replicas can then be created in the usual fashion, with all following the
locality setting of the master node.  This flag will also work with setting up
an initial replication network:

.. code-block:: bash

    trove create my_repl_set 7 --size 1 --locality anti-affinity --replica_count 3

If adding replicas to an existing set, an exception will be thrown as this flag
cannot be changed once it has been associated with an instance.  For example,
the following command would fail:

.. code-block:: bash

    trove create my_replica 7 --size 1 --locality affinity --replica_of <id>

The show command will also display the locality value:

.. code-block:: bash

    > trove show myinst

    +-------------------+-------------------------+
    | Property          | Value                   |
    +-------------------+-------------------------+
    | created           | 2015-12-13T12:36:59     |
    | datastore         | mysql                   |
    | datastore_version | 5.6                     |
    | flavor            | 7                       |
    | id                | <id>                    |
    | ip                | 10.64.151.6             |
    | name              | myinst                  |
    | status            | ACTIVE                  |
    | updated           | 2015-12-13T12:37:03     |
    | volume            | 1                       |
    | volume_used       | 0.1                     |
    | locality          | affinity                |
    +-------------------+-------------------------+

Internal API
------------

The Task Manager is responsible for creating the replication instances, so it
will need to be aware of the locality flag.  The relevant methods will be
changed to include this, as described above.

Once the flag is converted to a server group, a 'hint' will be created to pass
to the Nova client.  The converted hint data will be equivalent to the
corresponding ReST API values:

.. code-block:: python

    "os:scheduler_hints":
    {
        "group": "<id>"
    }


Guest Agent
-----------

Since the server group must be created before the Nova instance is created,
there are no anticipated Guest Agent changes.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  peterstac

Milestones
----------

Target Milestone for completion:
    Mitaka-3

Work Items
----------

The work will be undertaken with the following tasks:

    * Client (Python and CLI) changes
    * Server (API and TaskManager) changes


Upgrade Implications
====================

Since this change is completely backwards compatible, no upgrade issues are
expected.


Dependencies
============

None


Testing
=======

The replication scenario test will be modified to use --locality=affinity and
the results verified (i.e. that the Trove 'show' command returns the right
value for the 'locality' attribute).  A negative test with anti-affinity will
also be created (since devstack runs on one hypervisor, this test should always
fail to create replicas).


Documentation Impact
====================

This is a net-new feature, and as such will require documentation.


References
==========

.. [1] Output from running 'nova help server-group-create'


Appendix
========

None
