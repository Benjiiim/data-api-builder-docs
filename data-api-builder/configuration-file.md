---
title: Data API builder configuration file
description: This document assists in defining the configuration file in Data API builder.
author: anagha-todalbagi
ms.author: atodalbagi
ms.service: data-api-builder
ms.topic: configuration-file
ms.date: 04/06/2023
---

# Configuration file

## Summary

Data API builder configuration file contains the information to

+ Define the backend database and the related connection info
+ Define global/runtime configuration
+ Define what entities are exposed
+ Define authentication method
+ Define the security rules needed to access those identities
+ Define name mapping rules
+ Define relationships between entities (if not deducible from the underlying database)
+ Define specific behavior related to the chosen backend database

using the minimum amount of code.

## Environments support

Data API builder configuration file is able to support multiple environments, following the same behavior offered by ASP.NET Core for the `appSettings.json` file, as per: [Default Configuration](/aspnet/core/fundamentals/configuration/?view=aspnetcore-6.0#default-configuration&preserve-view=true). For example, the following two files represent a "base" and "environment specific" configuration files:

1. dab-config.json (base)
2. dab-config.Development.json (environment specific)

You must set the environment variable `DAB_ENVIRONMENT` to designate which environment file should be chosen Data API builder.

> [!NOTE]
> Environment specific configuration files override property values set in the base configuration file. For example, if the proprety `connection-string` is set in both `dab-config.json` and the environment-specific file, the environment specific configuration value is used.
>
> To learn more about using multiple configuration files together, see [here](../data-api-builder/data-api-builder-cli.md#using-data-api-builder-with-two-configuration-files)

## Setting environment variables

To use the environment variables we need to first set it up. There are two ways to do it:
1. Set the variables directly in the system.
2. Create an `.env` file listing the environment variable key-value pairs one line each and place this file in the same directory as the configuration file.

Setting your variables in .env file is a more convinient way to use and maintain environment variables.

For Example:
```txt
my-connection-string="Server=tcp:127.0.0.1,1433;Persist Security Info=False;User ID=USER_NAME;Password=PASSWORD;MultipleActiveResultSets=False;Connection Timeout=5;"
DAB_ENVIRONMENT=Development
```
By using the above .env  file, you can easily define and access these environment variables in the configuration file using the `@env()` function as mentioned below.

## Accessing environment variables

To avoid storing sensitive data into the configuration file itself, a developer can use the `@env()` function to access environment data. `env()` can be used anywhere a scalar value is needed. For example:

```json
{
  "connection-string": "@env('my-connection-string')"
}
```

## File structure

### Schema

The configuration file has a `$schema` property as the first property in the config to explicit the [JSON schema](https://code.visualstudio.com/Docs/languages/json#_json-schemas-and-settings) to be used for validation.

```json
"$schema": "..."
```

From versions 0.3.7-alpha and up, the schema file is available at the following location:

```https
GET https://github.com/Azure/data-api-builder/releases/download/<VERSION>-<suffix>/dab.draft.schema.json
```

make sure to replace the **VERSION-suffix** placeholder with the version you want to use, for example:

```https
GET https://github.com/Azure/data-api-builder/releases/download/v0.3.7-alpha/dab.draft.schema.json
```

If there are no suffix, then ignore it, for example:

```https
GET https://github.com/Azure/data-api-builder/releases/download/v0.5.35/dab.draft.schema.json
```

the **latest** version of the schema is always available at

```https
GET https://github.com/Azure/data-api-builder/releases/latest/download/dab.draft.schema.json
```

### Data source

The `data-source` element contains the information needed to connect to the backend database.

```json
"data-source" {
  "database-type": "",
  "connection-string": ""
}
```

`database-type` is a `enum string` and is used to specify what is the used backend database. Allowed values are:

+ `mssql`: for Azure SQL DB, Azure SQL MI and SQL Server
+ `postgresql`: for PostgreSQL
+ `mysql`: for MySQL
+ `cosmosdb_nosql`: for Cosmos DB NoSQL API
+ `cosmosdb_postgresql`: for Cosmos DB PostgreSQL API

while `connection-string` contains the ADO.NET connection string that Data API builder uses to connect to the backend database

### Runtime global settings

This section contains options that affect the runtime behavior and/or all exposed entities.

```json
"runtime": {
  "rest": {
    "path": "/api",
    "enabled": true
  },
  "graphql": {
    "path": "/graphql",
    "enabled": true
  },
  "host": {
    "mode": "<production> | <development>",
    "cors": {
      "origins": ["<array-of-strings>"],
      "credentials": false
    },
    "authentication":{
      "provider": "<StaticWebApps> | <AppService> | <AzureAD> | <Simulator>",
      "jwt": {
        "audience": "<Client_ID>",
        "issuer": "<Identity_Provider_Issuer_URL>"
      }
    }
  }
}
```

#### REST

`path`: defines the URL path where all exposed REST endpoints are available. For example if set to `/api`, the REST endpoint is exposed `/api/<entity>`. No sub-paths allowed. Optional. Default is `/api`.

`enabled`: Boolean flag that defines whether we want to enable to disable the REST endpoints globally. If disabled globally, no entities would be accessible via REST requests irrespective of the individual entity settings.

#### GraphQL

`path`: defines the URL path where the GraphQL endpoint is made available. For example if set to `/graphql`, the GraphQL endpoint is exposed `/graphql`. No sub-paths allowed. Optional. Default is `graphql`. Currently, a customized path value for GraphQL endpoint isn't supported.

`enabled`: Boolean flag that defines whether we want to enable to disable the GraphQL endpoints globally. If disabled globally, no entities would be accessible via GraphQL requests irrespective of the individual entity settings.

#### Host

`mode`: Define if the engine should run in `production` mode or in `development` mode. Only when running in development mode the underlying database errors are exposed in detail. Optional. Default value is `production`. With `production` mode, the default `--LogLevel` is `Error` whereas with `development` mode it's `Debug`. These default log levels can be overridden by starting the engine through `dab` CLI as mentioned [here](./run-using-data-api-builder-cli.md#run-engine-using-dab-cli).

`cors`: CORS configuration

`cors.origins`: Array with a list of allowed origins.

`cors.credentials`: Set [`Access-Control-Allow-Credentials`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials) CORS header. By default, it's `false`.

`authentication`: Configure the authentication process.

`authentication.provider`: What authentication provider is used. The supported values are `StaticWebApps`, `AppService` or `Azure AD`.

`authentication.provider.jwt`: Needed if authentication provider is `Azure AD`. In this section, you have to specify the `audience` and the `issuer` to allow the received JWT token to be validated and checked against the `Azure AD` tenant you want to use for authentication

### Entities

The `entities` section is where mapping between database objects to exposed endpoint is done, along with properties mapping and permission definition.

Each exposed entity is enclosed in a dedicated section. The property name is used as the name of the entity to be exposed. For example

```json
"entities" {
  "User": {
    ...
  }
}
```

instructs Data API builder to expose a GraphQL entity named `User` and a REST endpoint reachable via `/User` url path.

Within the entity section, there are feature specific sections:

#### GraphQL settings

##### GraphQL type

The `graphql` property defines the name with which the entity is exposed as a GraphQL type, if that is different from the entity name:

```json
"graphql":{
  "type": "my-alternative-name"
}
```

or, if needed

```json
"graphql":{
  "type": {
    "singular": "my-alternative-name",
    "plural": "my-alternative-name-pluralized"
  }
}
```

which instructs Data API builder runtime to expose the GraphQL type for the related entity and to name it using the provided type name. `plural` is optional and can be used to tell Data API builder the correct plural name for that type. If omitted Data API builder tries to pluralize the name automatically, following the English rules for pluralization (for example: https://engdic.org/singular-and-plural-noun-rules-definitions-examples)

##### GraphQL operation

The `graphql` element contains the `operation` property only for stored-procedures. The `operation` property defines the GraphQL operation that is configured for the stored procedure. It can be one of `Query` or `Mutation`.

For example:

```json
  {
    "graphql": "true",
    "operation": "query"
  }
```

instructs the engine that the stored procedure is exposed for graphQL through `Query` operation.

#### REST settings

##### REST path

The `path` property defines the endpoint through which the entity is exposed for REST APIs, if that is different from the entity name:

```json
"rest":{
  "path": "/entity-path"
}
```

##### REST methods

The `methods` property is only valid for stored procedures. This property defines the REST HTTP actions that the stored procedure is configured for.

For example:

```json
"rest":{
  "path": "/entity-path",
  "methods": [ "GET", "POST" ]
}

```

instructs the engine that GET and POST actions are configured for this stored procedure.  

#### Database object source

The `source` property tells Data API builder what is the underlying database object to which the exposed entity is connected to.

The simplest option is to specify just the name of the table or the collection:

```json
{
  "source": "dbo.users"
}
```

a more complete option is to specify the full description of the database if that isn't a table or a collection:

```json
{
  "source": {
    "object": "<string>",
    "type": "<view> | <stored-procedure> | <table>",
    "key-fields": ["<array-of-strings>"],
    "parameters": {
        "<name>": "<value>",
        "<...>": "<...>"
    }        
  }
}
```

where

+ `object` is the name of the database object to be used.
+ `type` describes if the object is a table, a view or a stored procedure.
+ `key-fields` is a list of columns to be used to uniquely identify an item. Needed if type is `view` or if type is `table` and there's no Primary Key defined on it.
+ `parameters` is optional and can be used if type is `stored-procedure`. The key-value pairs specified in this object will be used to supply values to stored procedures parameters, in case those aren't specified in the HTTP request.

More details on how to use Views and Stored Procedure in the related documentation [Views and Stored Procedures](./views-and-stored-procedures.md)

#### Relationships

The `relationships` section defines how an entity is related to other exposed entities, and optionally provides details on what underlying database objects can be used to support such relationships. Objects defined in the `relationship` section are exposed as GraphQL field in the related entity. The format is the following:

```json
"relationships": {
  "<relationship-name>": {
    "cardinality": "<one> | <many>",
    "target.entity": "<entity-name>",
    "source.fields": ["<array-of-strings>"],
    "target.fields": ["<array-of-strings>"],
    "linking.[object|entity]": "<entity-or-db-object-name",
    "linking.source.fields": ["<array-of-strings>"],
    "linking.target.fields": ["<array-of-strings>"]
  }
}
```

##### One-To-Many relationship

Using the following configuration snippet as an example:

```json
"entities": {
  "Category": {
    "relationships": {
      "todos": {
        "cardinality": "many",
        "target.entity": "Todo",
        "source.fields": ["id"],
        "target.fields": ["category_id"]
      }
    }
  }
}
```

the configuration is telling Data API builder that the exposed `category` entity has a One-To-Many relationship with the `Todo` entity (defined elsewhere in the configuration file) and so the resulting exposed GraphQL schema (limited to the `Category` entity) should look like the following:

```graphql
type Category
{
  id: Int!
  ...
  todos: [TodoConnection]!
}
```

`source.fields` and `target.fields` are optional and can be used to specify which database columns are used to create the query behind the scenes:

+ `source.fields`: database fields in the *source* entity (`Category` in the example) that are used to connect to the related item in the `target` entity
+ `target.fields`: database fields in the *target* entity (`Todo` in the example) that are used to connect to the related item in the `source` entity

These are optional if there's a Foreign Key constraint on the database between the two tables that can be used to infer that information automatically.

##### Many-To-One relationship

Similar to the One-To-Many but cardinality is set to `one`. Using the following configuration snippet as an example:

```json
"entities": {
  "Todo": {
    "relationships": {
      "category": {
        "cardinality": "one",
        "target.entity": "Category",
        "source.fields": ["category_id"],
        "target.fields": ["id"]
      }
    }
  }
}
```

the configuration is telling Data API builder that the exposed `Todo` entity has a Many-To-One relationship with the `Category` entity (defined elsewhere in the configuration file) and so the resulting exposed GraphQL schema (limited to the `Todo` entity) should look like the following:

```graphql
type Todo
{
  id: Int!
  ...
  category: Category
}
```

`source.fields` and `target.fields` are optional and can be used to specify which database columns are used to create the query behind the scenes:

+ `source.fields`: database fields in the *source* entity (`Todo` in the example) that are used to connect to the related item in the `target` entity
+ `target.fields`: database fields in the *target* entity (`Category` in the example) that are used to connect to the related item in the `source` entity

These are optional if there's a Foreign Key constraint on the database between the two tables that can be used to infer that information automatically.

##### Many-To-Many relationship

A many-to-many relationship is configured in the same way the other relationships type are configured, with the additional information about the association table or entity used to create the M:N relationship in the backend database.

```json
"entities": {
  "Todo": {
    "relationships": {
      "assignees": {
        "cardinality": "many",
        "target.entity": "User",
        "source.fields": ["id"],
        "target.fields": ["id"],
        "linking.object": "s005.users_todos",
        "linking.source.fields": ["todo_id"],
        "linking.target.fields": ["user_id"]
      }
    }
  }
}
```

the `linking` prefix in elements identifies those elements used to provide association table or entity information:

+ `linking.object`: the database object (if not exposed via Hawaii) that is used in the backend database to support the M:N relationship
+ `linking.source.fields`: database fields, in the *linking* object (`s005.users_todos` in the example), that is used to connect to the related item in the `source` entity (`Todo` in the sample)
+ `linking.target.fields`: database fields, in the *linking* object (`s005.users_todos` in the example), that is used to connect to the related item in the `target` entity (`User` in the sample)

The expected GraphQL schema generated by the above configuration is something like:

```graphql
type User
{
  id: Int!
  ...
  todos: [TodoConnection]!
}

type Todo
{
  id: Int!
  ...
  assignees: [UserConnection]!
}
```

#### Permissions

The section `permissions` defines who (in terms of roles) can access the related entity and using which actions. Actions are the usual CRUD operations: `create`, `read`, `update`, `delete`.

```json
{
  "permissions": [
    {
      "role": "...",
      "actions": ["create", "read", "update", "delete"],
      }
  ]
}
```

##### Roles

The `role` string contains the name of the role to which the defined permission applies.

```json
{
  "role": "reader"
}
```

##### Actions

The `actions` array details what actions are allowed on the associated role. When the entity is either a table or view, roles can be configured with a combination of the actions: `create`, `read`, `update`, `delete`.

The following example tells Data API builder that the contributor role permits the `read` and `create` actions on the entity:

```json
{
  "role": "contributor",
  "actions": ["read", "create"]
}
```

In case all actions are allowed, the wildcard character `*` can be used as a shortcut to represent all actions supported for the type of entity:

```json
{
  "role": "editor",
  "actions": ["*"]
}
```

For stored procedures, roles can only be configured with the `execute` action or the wildcard `*`. The wildcard `*` expands to the `execute` action for stored procedures.
For tables and views, the wildcard `*` action expands to the actions `create, read, update, delete`.

##### Fields

Role configuration supports granularly defining which database columns (fields) are permitted access in the section `fields`:

```json
{
  "role": "read-only",
  "action": "read",
  "fields": {
    "include": ["*"],
    "exclude": ["field_xyz"]
  }
}
```

That indicates to Data API builder that the role *read-only* can `read` from all fields except from `field_xyz`.

Both the simplified and granular `action` definitions can be used at the same time. For example, the following configuration limits the `read` action to specific fields, while implicitly allowing the `create` action to operate on all fields:

```json
{
  "role": "reader",
  "actions": [
    {
      "action": "read",
      "fields": {
        "include": ["*"],
        "exclude": ["last_updated"]
      }
    },
    "create"
  ]
}
```

In the `fields` section above, the wildcard `*` in the `include` section indicates all fields. The fields noted in the `exclude` section have precedence over fields noted in the `include` section. The definition translates to *include all fields except for the field 'last_updated'*.

##### Policies

The `policy` section, defined per `action`, defines item-level security rules (database policies) which limit the results returned from a request. The sub-section `database` denotes the database policy expression that is evaluated during request execution.

```json
  "policy": {
    "database": "<Expression>"
  }
```

- `database` policy: an OData expression that is translated into a query predicate that will be evaluated by the database.
  - for example The policy expression `@item.OwnerId eq 2000` is translated to the query predicate `WHERE Table.OwnerId  = 2000`

> A *predicate* is an expression that evaluates to TRUE, FALSE, or UNKNOWN. Predicates are used in the search condition of [WHERE](/sql/t-sql/queries/where-transact-sql) clauses and [HAVING](/sql/t-sql/queries/select-having-transact-sql) clauses, the join conditions of [FROM](/sql/t-sql/queries/from-transact-sql) clauses, and other constructs where a Boolean value is required.
([Microsoft Learn Docs](/sql/t-sql/queries/predicates?view=sql-server-ver16&preserve-view=true))

In order for results to be returned for a request, the request's query predicate resolved from a database policy must evaluate to `true` when executing against the database.

Two types of directives can be used when authoring a database policy expression:

- `@claims`: access a claim within the validated access token provided in the request.
- `@item`: represents a field of the entity for which the database policy is defined.

> [!NOTE]
> When Azure Static Web Apps authentication (EasyAuth) is configured, a limited number of claims types are available for use in database policies: `identityProvider`, `userId`, `userDetails`, and `userRoles`. See Azure Static Web App's [Client principal data](/azure/static-web-apps/user-information?tabs=javascript#client-principal-data) documentation for more details.

For example, a policy that utilizes both directive types, pulling the UserId from the access token and referencing the entity's OwnerId field would look like:

```json
  "policy": {
    "database": "@claims.UserId eq @item.OwnerId"
  }
```

Data API builder compares the value of the `UserId` claim to the value of the database field `OwnerId`. The result payload only includes records that fulfill **both** the request metadata and the database policy expression.

##### Limitations

Database policies are supported for tables and views. Stored procedures can't be configured with policies.

Database policies can't be used to prevent a request from executing within a database. This is because database policies are resolved as query predicates in the generated database queries and are ultimately evaluated by the database engine.

Database policies are only supported for the `actions` **create**, **read**, **update**, and **delete**.

Database policy OData expression syntax only supports:

- Binary operators [BinaryOperatorKind - Microsoft Learn](/dotnet/api/microsoft.odata.uriparser.binaryoperatorkind?view=odata-core-7.0&preserve-view=true) such as `and`, `or`, `eq`, `gt`, `lt`, and more.
- Unary operators [UnaryOperatorKind - Microsoft Learn](/dotnet/api/microsoft.odata.uriparser.unaryoperatorkind?view=odata-core-7.0&preserve-view=true) such as the negate (`-`) and `not` operators.
- Entity field names must "start with a letter or underscore, followed by at most 127 letters, underscores or digits" per [OData Common Schema Definition Language Version 4.01](https://docs.oasis-open.org/odata/odata-csdl-json/v4.01/odata-csdl-json-v4.01.html#sec_SimpleIdentifier)
    - Fields which do not conform to the mentioned restrictions can't be referenced in database policies. As a workaround, configure the entity with a `mappings` section to assign conforming aliases to the fields.

#### Mappings

The `mappings` section enables configuring aliases, or exposed names, for database object fields. The configured exposed names apply to both the GraphQL and REST endpoints. For entities with GraphQL enabled, the configured exposed name **must** meet GraphQL naming requirements. [GraphQL - October 2021 - Names ](https://spec.graphql.org/October2021/#sec-Names)

The format is: `<database_field>: <entity_field>`

For example:

```json
  "mappings": {
    "sku_title": "title",
    "sku_status": "status"
  }
```

means the `sku_title` field in the related database object is mapped to the exposed name `title` and `sku_status` is mapped to `status`. Both GraphQL and REST require using `title` and `status` instead of `sku_title` and `sku_status` and will additionally use those mapped values in all response payloads.

#### Sample config

This is a sample config file to give an idea of how the json config consumed by Data API builder might look like:

```json
{
  "$schema": "https://github.com/Azure/data-api-builder/releases/download/v{dab-version}/dab.draft.schema.json",
  "data-source": {
    "database-type": "mssql",
    "connection-string": "Server=localhost;Database=PlaygroundDB;User ID=PlaygroundUser;Password=ReplaceMe;TrustServerCertificate=false;Encrypt=True"
  },
  "mssql": {
    "set-session-context": true
  },
  "runtime": {
    "rest": {
      "enabled": true,
      "path": "/api"
    },
    "graphql": {
      "allow-introspection": true,
      "enabled": true,
      "path": "/graphql"
    },
    "host": {
      "mode": "development",
      "cors": {
        "origins": [],
        "allow-credentials": false
      },
      "authentication": {
        "provider": "StaticWebApps"
      }
    }
  },
  "entities": {
    "Author": {
      "source": "authors",
      "rest": false,
      "graphql": true,
      "permissions": [
        {
          "role": "anonymous",
          "actions": [ "*" ]
        }
      ]
    },
    "Book": {
      "source": "books",
      "rest": false,
      "graphql": true,
      "permissions": [
        {
          "role": "anonymous",
          "actions": [ "*" ]
        }
      ]
    }
  }
}
```
