.. meta::
   :description: Adding remote schema relationships with Postgres tables in Hasura
   :keywords: hasura, docs, remote schema relationship, remote join, remote schema, data federation

.. _pg_remote_schema_relationships:

Postgres: Remote schema relationships
=====================================

.. contents:: Table of contents
  :backlinks: none
  :depth: 2
  :local:

Introduction
------------

Remote schema relationships extend the concept of joining data across tables, to joining across tables and remote data sources. Once you create relationships between
types from your database and types created from APIs, you can then "join" them by running GraphQL queries.

These APIs can be custom GraphQL servers you write, third party SaaS APIs, or even other Hasura instances.

Because Hasura is meant to be a GraphQL server that you can expose directly to your apps, Hasura also handles security and authorization while providing remote joins.

.. note::

  To see example use cases, check out this `blog post <https://hasura.io/blog/remote-joins-a-graphql-api-to-join-database-and-other-data-sources/>`__.

.. admonition:: Supported from

  Remote schema relationships are supported from versions ``v1.3.0`` and above.

Create remote schema relationships
----------------------------------

Step 0: Add a remote schema
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Add a remote schema as described :ref:`here <adding_schema>`, if the schema isn't already added.

Step 1: Define and create the relationship
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The following fields can be defined for a remote schema relationship:

- **Name**: Define a name for the relationship.
- **Remote Schema**: Select a remote schema among all the ones you've created.
- **Configuration**: Set up the join configuration, to inject values as input arguments of the remote schema field.

  - **From column**: Input injected from table column values.
  - **From static value**: Input injected from a static value of your choice.

For this example, we assume that our schema has a ``users`` table with the fields ``name`` and ``auth0_id``.

.. rst-class:: api_tabs
.. tabs::

  .. tab:: Console

    - Head to the ``Data -> [table-name] -> Relationships`` tab.
    - Click the ``Add a remote relationship`` button.

    .. thumbnail:: /img/graphql/core/schema/add-remote-schema-relationship.png
       :alt: Opening the remote relationship section
       :width: 1000px

    - Define the relationship and hit ``Save``.

    .. thumbnail:: /img/graphql/core/schema/define-remote-schema-relationship.png
      :alt: Defining the relationship
      :width: 800px

  .. tab:: CLI

    You can add a remote schema relationship by adding it to the ``tables.yaml`` in the ``metadata`` directory:

    .. code-block:: yaml
       :emphasize-lines: 4-13

        - table:
            schema: public
            name: users
          remote_relationships:
          - definition:
              remote_field:
                auth0profile:
                  arguments:
                    id: $auth0_id
              hasura_fields:
              - auth0_id
              remote_schema: auth0
            name: auth0

    Apply the metadata by running:

    .. code-block:: bash

      hasura metadata apply

  .. tab:: API

    You can add a remote schema relationship by using the :ref:`create_remote_relationship metadata API <create_remote_relationship>`:

    .. code-block:: http

      POST /v1/query HTTP/1.1
      Content-Type: application/json
      X-Hasura-Role: admin

      {
        "type": "create_remote_relationship",
        "args": {
          "name": "auth0_profile",
          "table": "users",
          "hasura_fields": [
            "auth0_id"
          ],
          "remote_schema": "auth0",
          "remote_field": {
            "auth0": {
              "arguments": {
                "auth0_id": "$auth0_id"
              }
            }
          }
        }
      }

In this example, we've added a remote schema which is a wrapper around `Auth0 <https://auth0.com/>`__'s REST API (see example
`here <https://github.com/hasura/graphql-engine/tree/master/community/boilerplates/remote-schemas/auth0-wrapper>`__).

1. We name the relationship ``auth0_profile``.
2. We select the ``auth0`` schema that we've added.
3. We set up the config to join the ``auth0_id`` input argument of our remote schema field to the ``auth0_id`` column of this table (in this case, the ``users`` table).

Step 2: Explore with GraphiQL
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In the GraphiQL tab, test out your remote schema relationship.

.. graphiql::
  :view_only:
  :query:
    query {
      users {
        name
        auth0_profile {
          nickname
          email
          last_login
        }
      }
    }
  :response:
    {
      "data": {
        "users": [
          {
            "name": "Daenerys Targaryen",
            "auth0_profile": {
              "nickname": "Stormborn",
              "email": "mother.of.dragons@unburnt.com",
              "last_login": "2019-05-19T01:35:48.863Z"
            }
          }
        ]
      }
    }

.. _remote_schema_relationship_permissions:

Remote schema relationship permissions
--------------------------------------

Remote schema relationship permissions are derived from the
:ref:`remote schema permissions <remote_schema_permissions>` defined for the role.
When a remote relationship cannot be derived, the remote relationship field will
not be added to the schema for the role.

Some of the cases in which a remote relationship cannot be derived are:

1. There are no remote schema permissions defined for the role.
2. The role doesn't have access to the field or types that are used by the
   remote relationship.

.. note::

   Remote relationship permissions apply only if remote schema permissions
   are enabled in graphql-engine.
