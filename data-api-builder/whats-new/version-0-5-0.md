---
title: What's new for version 0.5.0
description: Release notes with new features, bug fixes, and updates listed for the Data API builder version 0.5.0.
author: seesharprun
ms.author: sidandrews
ms.reviewer: jerrynixon
ms.service: data-api-builder
ms.topic: whats-new 
ms.date: 03/28/2024
---

# What's new in Data API builder version 0.5.0

The full list of release notes for this version is available on GitHub: <https://github.com/Azure/data-api-builder/releases/tag/v0.5.0-beta>.

## Public JSON Schema

The public JSON schema gives you support for "intellisense," if you're using an IDE like Visual Studio Code that supports JSON Schemas. The `basic-empty-dab-config.json` file in the `samples` folder has an example starting point when manually creating the `dab-config.json` file.

## Public `Microsoft.DataApiBuilder` NuGet

`Microsoft.DataApiBuilder` is now available as a public NuGet package [here](https://www.nuget.org/packages/Microsoft.DataApiBuilder) for ease of installation using dotnet tool as follows:

```bash
dotnet tool install --global Microsoft.DataApiBuilder
```

## New `execute` action for stored procedures in Azure SQL

A new `execute` action is introduced as the only allowed action in the `permissions` section of the configuration file only when a source type backs an entity of `stored-procedure`. By default, only `POST` method is allowed for such entities and only the GraphQL `mutation` operation is configured with the prefix `execute` added to their name. Explicitly specifying the allowed `methods` in the `rest` section of the configuration file overrides this behavior. Similarly, for GraphQL, the `operation` in the `graphql` section, can be overridden to be `query` instead. For more information, see [views and stored procedures](../views-and-stored-procedures.md).

## New `mappings` section

In the `mappings` section under each `entity`, the mappings between database object field names and their corresponding exposed field names are defined for both GraphQL and REST endpoints.

The format is:

`<database_field>: <entity_field>`

For example:

```json
  "mappings":{
    "title": "descriptions",
    "completed": "done"
  }
```

The `title` field in the related database object is mapped to `description` field in the GraphQL type or in the REST request and response payload.

## Support for Session Context in Azure SQL

To enable an extra layer of Security (for example, Row Level Security (RLS)), DAB now supports sending data to the underlying Sql Server database via SESSION_CONTEXT. For more details, please refer to this detailed document on SESSION_CONTEXT: [Runtime to Database Authorization](https://github.com/Azure/data-api-builder/blob/cc7ec4f5a12c3e0fe87e1452f8989199d0aba8e6/docs/runtime-to-database-authorization.md).  

## Support for filter on nested objects within a document in PostgreSQL

With PostgreSQL, you can now use the object or array relationship defined in your schema, which enables to do filter operations on the nested objects just like Azure SQL.

```graphql
query {
  books(filter: { series: { name: { eq: "Foundation" } } }) {
    items {
      title
      year
      pages
    }
  }
}
```

## Support scalar list in Cosmos DB NoSQL

The ability to query `List` of Scalars is now added for Cosmos DB.

Consider this type definition.

```graphql
type Planet @model(name:"Planet") {
    id : ID,
    name : String,
    dimension : String,
    stars: [Star]
    tags: [String!]
}
```

It's now possible to run a query that fetches a List such as

```graphql
query ($id: ID, $partitionKeyValue: String) {
    planet_by_pk (id: $id, _partitionKeyValue: $partitionKeyValue) {
        tags
    }
}
```

## Enhanced logging support using log level

- The default log levels for the engine when `host.mode` is `Production` and `Development` are updated to `Error` and `Debug` respectively.
- During engine start-up, for every column of an entity, information such as exposed field names and the primary key is logged. This behavior happens even type when the field mapping is autogenerated.
- In the local execution scenario, all the queries that are generated and executed during engine start-up are logged at `Debug` level.
- For every entity, relationship fields such as `source.fields`, `target.fields`, and `cardinality` are logged. If there are many-many relationships, `linking.object`, `linking.source.fields`, and `linking.target.fields` inferred from the database (or from config file) are logged.
- For every incoming request, the role and the authentication status of the request are logged.
- In CLI, the `Microsoft.DataAPIBuilder` version is logged along with the logs associated with the respective command's execution.

## Updated CLI

- `--no-https-redirect` option is added to `start` command. With this option, the automatic redirection of requests from `http` to `https` can be prevented.

- In MsSql, session context can be enabled using `--set-session-context true` in the `init` command. A sample command is shown in this example:

  ```shell
  dab init --database-type mssql --connection-string "Connection String" --set-session-context true
  ```
  
- Authentication details such as the provider, audience, and issuer can be configured using the options `--auth.provider`, `--auth.audience`, and `--auth.issuer.` in the `init` command. A sample command is shown in this sample:

  ```shell
  dab init --database-type mssql --auth.provider AzureAD --auth.audience "audience" --auth.issuer "issuer"
  ```
  
- User friendly error messaging when the entity name isn't specified.
