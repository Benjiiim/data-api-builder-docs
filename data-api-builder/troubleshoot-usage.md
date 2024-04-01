---
title: Data API builder usage troubleshooting guide
description: In this article, troubleshoot common problems that might occur when you're using Data API builder. 
author: yorek 
ms.author: damauri 
ms.service: data-api-builder 
ms.topic: troubleshooting-general 
ms.date: 02/27/2023 
---

# Troubleshoot Data API builder usage

This article provides solutions to common problems that might arise when you're using Data API builder.

## Generic Endpoint Errors

### HTTP 400 “Bad Request”

#### Invalid Data API builder endpoint
DAB returns an HTTP 400 bad request error when the base of the URL path component does not match a value set in your runtime configuration. The base of the URL path component maps to DAB's REST or GraphQL endpoint.

You configure the root path of the GraphQL and REST endpoints in the `runtime` section of DAB's runtime configuration.  For example, the following configuration sets `/api` as the root of the REST endpoint and `/graphql` as the root of the GraphQL endpoint:

```json
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
    "...": "...",
}
```

In other words, the previous runtime configuration defines what value you must use at the beginning of your path for the REST and GraphQL endpoints, respectively:

```https
/api/<entity>
```

```https
/graphql
```

> [!NOTE]
> When using Data API builder with Static Web Apps using the Database Connections feature, the beginning of the URL path is `/data-api`. You can then add the value for your desired endpoint after that value. For example, `/data-api/api/<entity>` for REST and `/data-api/graphql` for GraphQL.

#### Static Web Apps Database Connections configuration error

When you use Data API builder with the Static Web Apps Database Connections feature, you may encountered an HTTP 400 error with the primary message "Response status code does not indicate success: 400 (Bad Request)." in the resopnse body:

```json
{"Message":"{\u0022Message\u0022:\u0022Response status code does not indicate success: 400 (Bad Request).\u0022,\u0022ActivityId\u0022:\u0022<GUID>\u0022}","ActivityId":"<GUID>"}
```

This error may indicate an issue with the configuration you supplied when linking your database.

To troubleshoot:

1. Validate your DB credentials work in a tool like Azure Data Studio or SSMS.
1. Unlink/Relink the connected database.

If you still encounter the same error after troubleshooting, ensure that you select the Exceptions checkbox in the networking blade of your Azure Database resource to **"Allow Azure services and resources to access this server"**. The Azure Portal's description of this option:

> This option configures the firewall to allow connections from IP addresses allocated to any Azure service or asset, including connections from the subscriptions of other customers.

![Selected checkbox to allow Azure services and resources to access this server](./media/allow-azure-resources-to-access-server.png)

## REST endpoints

### HTTP 404 “Not Found” Errors

An HTTP 404 error is returned if the requested URL points to a route not associated with any entity. By default, the name of the entity is also the route name.  For example, if you've configured the sample `Todo` entity in the configuration file like in the following sample:

```json
"Todo": {
      "source": "dbo.todos",
      "...": "..."
    }
```

The entity `Todo` is reachable via the following route:

```https
/<rest-route>/Todo
```

If you've specified the `rest.path` property in the entity configuration to `todo` as shown in the following example:

```json
"Todo": {
    "source": "dbo.todos",
    "rest": {
      "path": "todo"
    },
    "...": "..."
  }
```

Then the URL route for the `Todo` entity is `todo` with all lower case characters matching the exact value defined in the runtime configuration:

```https
/<rest-route>/todo
```

## GraphQL endpoints

## HTTP 400 “Bad Request” Error

GraphQL requests will result in an HTTP 400 "Bad Request" error every time the GraphQL request is improperly constructed. It could be that a non-existing entity field is specified, or that the entity name is misspelled. Data API builder returns a descriptive error in the response payload with details about the error itself.

The response body of the returned error will state *"Either the parameter query or the parameter ID has to be set."* when you send a `GET` request to the GraphQL endpoint. Make sure you are sending your GraphQL requests using HTTP `POST`.

## HTTP 404 “Not Found” Error

Make sure the GraphQL request is sent using the HTTP POST method to the configured GraphQL endpoint. By default, the GraphQL endpoint route is `/graphql`.

## Error "The object type Query has to at least define one field in order to be valid."

The error message `The object type Query has to at least define one field in order to be valid.` is returned when Data API builder startup fails to generated a GraphQL schema based on your runtime configuration. The GraphQL specification requires at least one *Query* object be defined in a GraphQL schema. You must validate that your runtime config has one of the following:

- `read` action defined for at least one GraphQL enabled entity. GraphQL is enabled on entities by default, so ensure that you **aren't** applying this mitigation to an entity configured with `{"graphql": false}`.
- `{ "graphql": { "operation": query" } }` is defined on at least one stored procedure entity when you are only exposing stored procedures, which overrides the default stored procedure GraphQL operation type `mutation`.
  - You must have at least one stored procedure which only reads and doesn't modify data. Otherwise, GraphQL schema generation will fail due to an empty `query` field in the schema.

## Introspection doesn't work with my GraphQL endpoint

GraphQL schema introspection is utilized by tooling that supports GraphQL.

Make sure you set the runtime configuration property `allow-introspection` to `true` in the `runtime.graphql` configuration section. For example:

```json
"runtime": {
    "...": "...",
    "graphql": {
      "allow-introspection": true,
      "enabled": true,
      "path": "/graphql"
    },
    "...": "..."
}
```

## Error "The mutation operation <operation_name> was successful but the current user is unauthorized to view the response due to lack of read permissions"

For a graphQL mutation operation to receive a valid response, read permission should be configured in addition to the respective mutation operation type -  create/update/delete. As the error suggests, the mutation operation(create/update/delete) was successful at the database layer but the lack of read permission is causing Data API builder to return an error message. So, make sure to configure read permission either in the Anonymous role or the role with which you would like to execute the mutation operation. 

## General errors

## Request returns an HTTP 500 error

HTTP 500 errors indicate that Data API builder can't properly operate on the backend database. Make sure that

- Data API builder can still connect to the configured database
- The database objects used by Data API builder are still available and accessible

To allow DAB to return specific database errors in responses, you need to set the `runtime.host.mode` configuration property to `development`.

```json
"runtime": {
    [...]
    "host": {
        "mode": "development",
        [...]
    }
}
```

By default DAB runs with `runtime.host.mode`set to `production`. In `production` mode, Data API builder doesn't return detailed errors in the response payload.

In both `development` or `production` modes, DAB will write detailed errors to the console to help with troubleshooting.

## Unauthenticated and unauthorized requests

### HTTP 401 “Unauthorized” Errors

HTTP 401 errors occur when the endpoint and entity being accessed require authentication and the requestor does not supply valid authentication metadata with their request.

When you configure DAB to use Azure Active Directory authentication, your requests will result in HTTP 401 errors when the provided bearer (access) token is invalid. There are many reasons an access token may be invalid. A non-exhaustive list of reasons are:

- Expired access token
- Access token is not meant for DAB's API (wrong access token audience)
- Access token is not created from the expected authority (invalid access token issuer).

Make sure that you've generated a access token using the `issuer` value defined in the `authentication` section of the runtime config.
Make sure that your access token is generated for the `audience` value defined in the `authentication` section of the runtime config.

For example, in this sample configuration:

```json
"authentication": {
    "provider": "AzureAD",
    "jwt": {
        "issuer": "https://login.microsoftonline.com/24c24d79-9790-4a32-bbb4-a5a5c3ffedd5/v2.0/",
        "audience": "b455fa3c-15fa-4864-8bcd-88fd83d686f3"
    }
}
```

You must generate a token valid for the defined audience. When using the AZ CLI, you can get an access token by explicitly specifying the audience in the `resource` parameter:

```azurecli
az account get-access-token --resource "b455fa3c-15fa-4864-8bcd-88fd83d686f3"
```

For more information about Azure AD authentication configuration in Data API builder, see the article [Authentication](./authentication-azure-ad.md).

### HTTP 403 “Forbidden” Errors

If you're sending an authenticated request, either using Static Web Apps integration or Azure AD, you may receive the error HTTP 403 “Forbidden”. 

A HTTP 403 Forbidden will occur in the following circumstances: This error indicates that you're trying to use a role that hasn't been configured in the configuration file.

1. When you don't supply a `X-MS-API-ROLE` HTTP header specifying a role name. (This only applies when you intend to use a non-system role (not `authenticated` nor `anonymous`) because by default, **authenticated** requests execute in the context of the system role `authenticated`).
1. When the role you define in the `X-MS-API-ROLE` hasn't been configured in DAB's runtime configuration file.
1. When the role you define in the `X-MS-API-ROLE` header doesn't match a role in your access token.

For example, with the following runtime configuration file where the role `role1` is defined, you must supply a `X-MS-API-ROLE` HTTP header with the value `role1`.

> [!CAUTION]
> Roles name matching is case-sensitive

```json
"Todo": {
    "role": "role1",
    "actions": [ "*" ]
}
```
