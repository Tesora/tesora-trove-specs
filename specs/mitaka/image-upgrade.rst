..
    This work is licensed under a Creative Commons Attribution 3.0 Unported
    License.

    http://creativecommons.org/licenses/by/3.0/legalcode

    Sections of this template were taken directly from the Nova spec
    template at:
    https://github.com/openstack/nova-specs/blob/master/specs/template.rst

..


=============
Image Upgrade
=============

Trove needs to address how guest instances will update their database
software, the Trove guest agent, and the underlying operating system.
This blueprint proposes a technique which will allow guest instances
to be updated by migrating the data volume to a new instance based on
an updated guest image.

Launchpad Blueprint:
https://blueprints.launchpad.net/trove/+spec/image-upgrade


Problem Description
===================

Trove creates database images by creating a nova instance from a
provided operating system image, attaching a volume to the instance to
hold the user's data.  By using a known image with the database
software preconfigured, an operator can ensure that the user will
receive a known version of database software, configured with an
operating system certified to work with that version of database
software.  This technique elimanates many of the vaguaries of
provisioning applications, allowing an operator to provide a published
level of service to the database user.

Unfortunately, running computer instances aren't static over their
lifetimes.  Over time, it may be necessary for the instance to be
upgraded with security patches, new database software, or updates to
the Trove guest agent.  The cloud operator and the end user must be
provided with a way to migrate their databases to new operating
environments.

There are a number of changes which a database instance might be
subjected to over time, including:

* operating system security patches
* database security patches
* database version upgrades
* Trove (guest agent) upgrades
* Openstack version upgrades
* operating system upgrades


Proposed Change
===============

The traditional approach to system upgrades is to simply execute
system utilities such as "apt-get update" to upgrade all or parts of a
running system.  For large installations, this technique could be
automated by tools such as Ansible or Salt to apply the changes
automatically to many systems.  This approach isn't ideal for
database-as-a-service as there is little control over which components
are upgraded, and to which version; an update could leave a database
instance in a state where the database does not operate properly, or
could even cause the data itself to become corrupted.

It is crucial that Trove develop a method to update instances which
provides a high probability of leaving the database in a safe,
consistent, and useable state.

To ensure a safe, consistent state, this blueprint proposes a
technique for upgrading an instance which transitions it from one
known image to a new known image running updated software.  The full
state of each instance state will be known.  At it's core, this
technique will involve detaching the cinder volume containing the
database data and attaching it to a new instance based on a different
(newer) image.

The bulk of the "variable" data on a Trove instance exists on the
attached volume where the database stores its data.  There is also
some operating system state which must be preserved during this
transition, consisting primarily of networking data (the instance name
and ip address), some Trove configuration data (guest_info.conf and
trove-guestagent.conf), and the database configuration data (such as
/etc/mysql).

The core of this technique will be based on the rebuild API provided
by Nova.  The Nova rebuild function upgrades a running instance to a
new image, ensuring that the networking configuration is applied to
the new instance.  The newly created instance will come up with the
same network configuration (ip, hostname, and Neutron configuration)
as the instance from which it was created.  Trove guest functionality
will be added to allow the guest to capture additional state about the
instance (primarily database configuration), and re-apply that state
to the system after rebuild.  The result will be a running,
connectable database provisioned on a new instance running updated
software.

The selection of image to upgrade to will be based on the current
datastore_version mechanism.  Essentially, the user will be allowed to
upgrade an instance to a newer datastore_version of its datastore.
Safeguards will be put in place to ensure that the user is selecting
an appropriate datastore_version to upgrade to; for example, the user
would only be allowed to upgrade a mysql instance from
datastore_version 5.5 to 5.6, not to 5.0 or 6.7.  This constraint
functionality may be postponed to a future release.

If your specification proposes any changes to the Trove REST API such
as changing parameters which can be returned or accepted, or even
the semantics of what happens when a client calls into the API, then
you should add the APIImpact flag to the commit message. Specifications with
the APIImpact flag can be found with the following query:

https://review.openstack.org/#/q/status:open+project:openstack/trove-specs+message:apiimpact,n,z


Configuration
-------------

No configuration changes are planned.

Database
--------

A future blueprint will outline the changes necessary to support the
upgrade constraints on datastore_versions, but this specification will
be limited to implementing the upgrade functionality without the
constraints.

Public API
----------

A new REST API call will be added in support of this functionality.

REST API: PATCH /instances/<instance id>
REST body:

.. code-block:: json

    {
        "instance": {
            "datastore_version": "<datastore_version_uuid>"
        }
    }

REST result:

.. code-block:: json

    {}

REST return codes:

    202 - Accepted.
    400 - BadRequest. Server could not understand request.
    404 - Not Found. <datastore_version_id> not found.

Public API Security
-------------------

There are no envisioned security implications.

Python API
----------

A new method will be implemented in the trove API.  This method will
upgrade a instance to the image specified by the provided
datastore_version.

.. code-block:: python

    upgrade(instance, datastore_version)

:instance: the instance to upgrade
:datastore_version: the datastore version, or its id, to which the
                    trove instance will be upgraded


CLI (python-troveclient)
------------------------

A new CLI call will be implemented.  This new wcall will upgrade a
instance to the image specified by the provided datastore_version.

.. code-block:: bash

    trove upgrade <instance> <datastore_version>

:instance: the instance to upgrade
:datastore_version: the datastore version to which the instance will
                    be upgraded

Internal API
------------

A new method will be added in support of this functionality.

.. code-block:: python

    def upgrade(self, instance_id, datastore_version_id):
        LOG.debug("Making async call to upgrade guest to %s "
                  % datastore_version_id)

        cctxt = self.client.prepare(version=self.version_cap)
        cctxt.cast(self.context, "upgrade", instance_id=instance_id,
                   datastore_version_id=datastore_version_id)


Guest Agent
-----------

Two new operations will be implemented in the guest agent API.  It is
expected that each datastore will (optionally) override these methods
to implement any needed functionality before and after the image
upgrade proceeds.  Mysql, for example, would use these methods to copy
its configuration data from /etc/mysql to the data volume before the
image upgrade, copying them back and restarting the mysql server after
the image upgrade.

It is expected that the pre_upgrade method will validate that it is
possible to perform the requested upgrade; for example, there may be a
configuration override specified for an instance which is not
compatible with the new datastore_version.  In the event that an
upgrade cannot be performed, the pre_upgrade method will raise an
exception - any exception will cause the taskmanager to abort the
upgrade process for that instance.

.. code-block:: python

    def pre_upgrade(self):
        """Prepare the guest for upgrade."""
        LOG.debug("Sending the call to prepare the guest for upgrade.")
        self._call("pre_upgrade", AGENT_HIGH_TIMEOUT, self.version_cap)

    def post_upgrade(self):
        """Recover the guest after upgrading the guest's image."""
        LOG.debug("Recover the guest after upgrading the guest's image.")
        self._call("post_upgrade", AGENT_HIGH_TIMEOUT, self.version_cap)


Alternatives
------------

The alternative to upgrading images is to upgrade via apt-get or yum
in the running instance.  While this is the normal procedure for
upgrading instances, it has undesired implications for Trove.  Trove
aims to provide a known service level, but upgrading a running
instance has the potential to leave an instance in an unknown state.
There could also be issues around installations which don't allow
trove instances to access the internet directly as trove would have to
provide some mechanism for delivering all of the deb/rpm packages
required to update an instance.


Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a spec where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please
designate the primary author and contact.

Primary assignee:
  <launchpad-id or None>

Can list additional ids if they intend on doing substantial
implementation work on this spec.

Milestones
----------

Target Milestone for completion:
    Mitaka

Work Items
----------

This feature has been already been prototyped.  The work required to
bring the prototype in line with this spec is:

* The prototype uses image id rather than datastore_version
* add trove.upgrade.start/end/error notifications
* implement unit tests
* investigate if int-tests need to be updated for this feature
* document the upgrade procedure

Upgrade Implications
====================

* eliminates or adds new notifications (events),

If the change has upgrade implications, also remember to:

* add the DocImpact keyword to the commit, and

* provide sufficient information in the commit message or in the
  documentation bug that gets created.

For more information about the DocImpact keyword, refer to
https://wiki.openstack.org/wiki/Documentation/DocImpact

Note: Documentation for the CLI commands are automatically generated
from the help strings when a new version of the CLI is released, so
a DocImpact keyword is not typically required for python-troveclient
changes.


Dependencies
============

n/a


Testing
=======

Unit tests will be added as appropriate.

We will investigate the addition of new int-tests.  This may be
limited to upgrading an instance to a new version of its own image as
it may not be possible to upgrade existing images to "newer" versions
of themselves.  This will however test the upgrade procedure and its
support in the guest agent (at least for mysql).



Documentation Impact
====================

What is the impact on the docs team of this change? Some changes might require
donating resources to the docs team to have the documentation updated. Don't
repeat details discussed above, but please reference them here.


References
==========

n/a

Appendix
========

n/a
