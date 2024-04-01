---
title: Host GraphQL endpoints
description: Review GraphQL endpoint hosting for Data API builder including, how to expose endpoints, schema definition, endpoint configuration, and invocation.
author: seesharprun
ms.author: sidandrews
ms.reviewer: jerrynixon
ms.service: data-api-builder
ms.topic: concept-article
ms.date: 04/01/2024
# Customer Intent: As a developer, I want to use the Data API Builder, so that I can host GraphQL endpoints.
---

# Host GraphQL endpoints in Data API builder

Entities configured to be available via GraphQL will be available at the default path

```text
http://<dab-server>/graphql
```

Data API builder automatically generates a GraphQL schema with query and mutation fields for all configured entities. The GraphQL schema can be explored using a modern GraphQL client like [Insomnia](http://insomnia.rest/) or [Postman](https://www.postman.com/), so that you have IntelliSense and autocomplete.

If you followed the [Getting Started](./get-started/get-started-with-data-api-builder.md) example, where there are the `books` and the `authors` entity configured for GraphQL access, you can see how easy is to use GraphQL.

## Resultset format

The returned result is a JSON object with this format:

```json
{
    "data": {}    
}
```

> [!NOTE]
> Only the first 100 items are returned by default.

## Supported GraphQL root types

Data API builder supports the following GraphQL root types:

[Queries](#queries)
[Mutations](#mutations)

## Queries

Each entity has support for the following actions:

- [Pagination](#pagination)
- [Query by Primary key](#query-by-primary-key)
- [Generic Query](#generic-query)

Data API builder, unless otherwise specified, uses the *singular* name of an entity whenever a single item is expected to be returned, and uses the *plural* name of an entity whenever a list of items is expected to be returned. For example, the `book` entity has:

- `book_by_pk()`: to return zero or one entity
- `books()`: to return a list of zero or more entities

### Pagination

All query types returning zero or more items support pagination: 

```graphql
{
  books
  {
    items {
      title
    }
    hasNextPage
    endCursor
  }
}
```

- the `item` object allows access to entity fields
- `hasNextPage` is set to true if there are more items to be returned
- `endCursor` returns an opaque cursor string that can be used with [`first` and `after`](#first-and-after) query parameters to get the next set (or page) of items.

### Query by Primary key

Every entity support retrieval of a specific item via its Primary Key, using the following query format:

```graphql
<entity>_by_pk(<pk_colum>:<pk_value>)
{
    <fields>
}
```

for example:

```graphql
{
  book_by_pk(id:1010) {
    title
  }
}
```

### Generic query

Every entity also supports a generic query pattern so that you can ask for  only the items you want, in the order you want, using the following parameters:

- [`filter`](#filter): filters the returned items
- [`orderBy`](#orderby): defines how the returned data is sorted
- [`first` and `after`](#first-and-after): returns only the top `n` items

for example:

```graphql
{
  authors(
    filter: {
        or: [
          { first_name: { eq: "Isaac" } }
          { last_name: { eq: "Asimov" } }
        ]
    }
  ) {
    items {
      first_name
      last_name
      books(orderBy: { year: ASC }) {
        items {
          title
          year
        }
      }
    }
  }
}
```

### `filter`

The value of the `filter` parameter is predicate expression (an expression that returns a boolean value) using entity's fields. Only items where the expression evaluates to 'True' are included in the response. For example:

```graphql
{
  books(filter: { title: { contains: "Foundation" } })
  {
    items {
      id
      title
      authors {
        items {
          first_name
          last_name
        }
      }
    }
  }
}
```

returns all the books with the word `Foundation` in the title.

The operators supported by the `filter` parameter are:

Operator                 | Description           | Example
--------------------     | --------------------- | -----------------------------------------------------
**Comparison Operators** |                       |
eq                       | Equal                 | `books(filter: { title: { eq: "Foundation" } })`
neq                      | Not equal             | `books(filter: { title: { neq: "Foundation" } })`
gt                       | Greater than          | `books(filter: { year: { gt: 1990 } })`
gte                      | Greater than or equal | `books(filter: { year: { gte: 1990 } })`
lt                       | Less than             | `books(filter: { year: { lt: 1990 } })`
lte                      | Less than or equal    | `books(filter: { year: { lte: 1990 } })`
isNull                   |                       | `books(filter: { year: { isNull: true} })`
**String Operators**     |                       |
contains                 | Contains              | `books(filter: { title: { contains: "Foundation" } })`
notContains              | Doesn't Contain       | `books(filter: { title: { notContains: "Foundation" } })`
startsWith               | Starts with           | `books(filter: { title: { startsWith: "Foundation" } })`
endsWith                 | End with              | `books(filter: { title: { endsWith: "Empire" } })`
**Logical Operators**    |                       |
and                      | Logical and           | `authors(filter: { and: [ { first_name: { eq: "Robert" } } { last_name: { eq: "Heinlein" } } ] })`
or                       | Logical or            | `authors(filter: { or: [ { first_name: { eq: "Isaac" } } { first_name: { eq: "Dan" } } ] })`

### `orderBy`

The value of the `orderby` set the order with which the items in the resultset are returned. For example:

```graphql
{
  books(orderBy: {title: ASC} )
  {
    items {
      id
      title
    }
  }
}
```

returns books ordered by `title`.

### `first` and `after`

The parameter `first` limits the number of items returned. For example:

```graphql
query {
  books(first: 5)
  {
    items {
      id
      title
    }
    hasNextPage
    endCursor
  }
}
```

returns the first five books. When no `orderBy` is specified, items are ordered based on the underlying primary key. The value provided to `orderBy` must be a positive integer.

If there are more items in the `book` entity than those requested via `first`, the `hasNextPage` field will evaluate to `true`, and the `endCursor` will return a string that can be used with the `after` parameter to access the next items. For example:

```graphql
query {
  books(first: 5, after: "W3siVmFsdWUiOjEwMDQsIkRpcmVjdGlvbiI6MCwiVGFibGVTY2hlbWEiOiIiLCJUYWJsZU5hbWUiOiIiLCJDb2x1bW5OYW1lIjoiaWQifV0=")
  {
    items {
      id
      title
    }
    hasNextPage
    endCursor
  }
}
```

## Mutations

For each entity, mutations to support create, update and delete operations are automatically created. The mutation operation is created using the following name pattern: `entity<operation>`. For example, for the `book` entity, the mutations would be:

- `createbook`: create a new book
- `updatebook`: update an existing book
- `deletebook`: delete the specified book

### Create

To create a new element of the desired entity, the `create<entity>` mutation is provided. The create mutation requires the `item` parameter, where values for entity's mandatory fields, to be used when creating the new item, are specified.

```graphql
create<entity>(item: <entity_fields>)
{
    <fields>
}
```

for example:

```graphql
mutation {
  createbook(item: {
    id: 2000,
    title: "Leviathan Wakes"    
  }) {
    id
    title
  }  
}
```

### Update

To create a new element of the desired entity, the `update<entity>` mutation is provided. The update mutation requires two parameters:

- `<primary_key>`, the key-value list of primary key columns and related values to identify the element to be updated
- `item`: parameter, with entity's mandatory fields values, to be used when updating the specified item

```graphql
update<entity>(<pk_colum>:<pk_value>, [<pk_colum>:<pk_value> ... <pk_colum>:<pk_value>,] item: <entity_fields>)
{
    <fields>
}
```

for example:

```graphql
mutation {
  updatebook(id: 2000, item: {
    year: 2011,
    pages: 577    
  }) {
    id
    title
    year
    pages
  }
}
```

### Delete

To delete a new element of the desired entity, the `delete<entity>` mutation is provided. The primary key of the element to be deleted is the required parameter.

```graphql
delete<entity>(<pk_colum>:<pk_value>, [<pk_colum>:<pk_value> ... <pk_colum>:<pk_value>,])
{
    <fields>
}
```

for example:

```graphql
mutation {
  deletebook(id: 1234)
  {
    id
    title
  }  
}
```

### Database transactions for a mutation

To process a typical GraphQL mutation request, Data API builder constructs two database queries. One of the database queries performs the update (or) insert (or) delete action that is associated with the mutation.
The other database query fetches the data requested in the selection set.

Data API builder executes both database queries in a transaction. Transactions are created only for SQL database types.

The following table lists the isolation levels with which the transactions are created for each database type.

|**Database Type**|**Isolation Level**|**Isolation Level Docs**
:-----:|:-----:|:-----|
Azure SQL (or) SQL Server|Read Committed|[Azure SQL docs](https://www.learn.microsoft.com/sql/t-sql/language-elements/transaction-isolation-levels)
MySQL|Repeatable Read|[MySQL docs](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read)
PostgreSQL|Read Committed|[PostgreSQL docs](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED)
