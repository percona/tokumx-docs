.. _audit:

=======
 Audit
=======

|TokuMX| can create a separate log of select server events, allowing administrators to audit the activity of different users throughout the lifetime of the process. Audit events range from coarse operations like create/drop commands to more fine grained events, like authorization checks done for CRUD/DML operations.

The audit log is stored in JSON, making it easy to parse and query in any custom application. Users can also create a filter for audit events that can ignore or focus on different event metadata, including type, users, namespaces, and more.

The Community Edition of |TokuMX| does not have audit capabilities.

Activation
==========
By default, auditing is disabled. It is activated from the command line on server startup. Auditing remains active until shutdown, it cannot be controlled dynamically.

The following parameters control auditing. See the audit parameters in :ref:`new_and_modified_server_parameters` for more details.

:variable:`auditDestination`

Mandatory

This is the type of log audit will create and use. In |TokuMX| v2.0, this can only be set to ``file``.

:variable:`auditFormat`

Mandatory

This is the audit event entry format. In |TokuMX| v2.0, this can only be set to JSON.

:variable:`auditPath`

Mandatory

This required for the ``file`` :variable:`auditDestination` type. It must be the fully qualified name to the file you wish |TokuMX| to create.

.. note::
  This file will rotate in the same manner as the system ``logpath``, either on server reboot or using the `logRotate <http://docs.mongodb.org/manual/reference/command/logRotate/>`_ command. The time of the rotation will be added to old file's name.

:variable:`auditFilter`

Optional

Specifies a filter to capture a subset of audit events. This should be a JSON string that is treated as a query object; each audit log event that matches this query will be logged, events which do not match this query will be ignored.

To learn more about how to filter events, see :ref:`schema` below.

Example:
For example, to log only events from a user named "tim", start the server with the following parameters:

.. code-block:: bash

  $ mongod                                \
  --auditDestination file                 \
  --auditFormat JSON                      \
  --auditPath /var/log/tokumx/audit.json  \
  --auditFilter '{ "users.user" : "tim" }'

.. _schema:

Schema
======

Each entry in the audit log is a JSON object. The form of these objects is identical to the events logged by `MongoDB Enterprise Audit <http://docs.mongodb.org/manual/reference/audit-message/>`_.

.. warning:: 
 In |TokuMX| v2.0, the authorization system is the same as in |MongoDB| v2.4, and therefore does not support custom roles. As a result, there are no audit events for the actions ``createRole``, ``updateRole``, ``dropRole``, ``dropAllRolesFromDatabase``, ``grantRolesToRole``, ``revokeRolesFromRole``, ``grantPrivilegesToRole``, and ``revokePrivilegesFromRole``.

.. _filtering:

Filtering
=========

A BSON filter can be provided on the command line to turn on or off logging of events. The filter can be scoped to anything in the `event schema <http://docs.mongodb.org/manual/reference/audit-message/>`_. This includes ``atype``, host and port settings, users, and dbs.

This allows administrators to limit the amount of logging to a select few events or users. It also enables administrators to ignore a small subset of events they do not want to see in the log, while still getting a good overall image of activity on the server.

Here are some examples of filter objects (and their associated effects) you can pass on the command line:

Logging a set of events:

.. code-block:: javascript

  --auditFilter '{ atype: { $in: ["createIndex", "dropIndex"] } }'

This will audit all ``createIndex`` and ``dropIndex`` events.

Excluding users:

.. code-block:: javascript

  --auditFilter '{ "users.user" : { $nin: ["john", "paul", "george"] } }'

This will audit all events by all users except John, Paul and George. We want to keep an eye on Ringo.

Production database only:

.. code-block:: javascript

  --auditFilter '{ "users.db" : "production" }'

This will audit all events, by all users, but only in the ``production`` db.

Combining fields

.. code-block:: javascript

  --auditFilter '{ atype: "createCollection", users: { db: "test", user:"testuser" } }'

This will only log ``createCollection`` events, on the ``test`` db, by user ``testuser``.

.. _durability:

Durability
==========

Whenever something occurs in the system that triggers an audit event, and the respective event matches :variable:`auditFilter`, the server writes that event to the audit log. There are no exceptions to this rule.

To avoid losing events on a crash, the server uses ``fsync()`` on the audit log file (when using the ``file`` :variable:`auditDestination`) after every write to that same audit log file. This guarantees durability of the event, even if the server can no longer write any information to its own logs from that point forward.

Without this constraint, an auditable event could cause some corrupt state or a fatal error, then crash/restart the server. Without audit enabled, the particular event or user may not be known at recovery because no relevant log data was flushed to disk before the crash/restart. There would be no record of that corrupting event or user. With fsync(), that offending event, if selected in the filter, will be available in the audit log after the crash/restart.

For example, basic |MongoDB| does not durably commit all audit log entries, and therefore `may lose events on a crash <http://docs.mongodb.org/manual/core/auditing/#audit-guarantee>`_.

.. warning:: 
 One caveat of this is that it can degrade performance. Some queries and operations that are normally very fast could be slowed down, if auditing is enabled.
 
 This is why it is very important to choose what events one wishes to audit in the audit filter. It is up to the administrator to strike a balance between the amount and type of information stored in the audit log, and the impact on operations that normally may not cause so much I/O.
