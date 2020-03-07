---
id: graphql-interface-union-types
title: Using GraphQL Interface And Union Types
sidebar_label: Interface and Union Types
---

## Overview

This page describes how interface and union types can be used with neo4j-graphql.js. 

GraphQL supports two kinds of abstract types: interfaces and unions. Interfaces are abstract types that include a set of fields that all implementing types must include. A union type indicates that a field can return one of several object types, but doesn't specify any fields that must be included in the implementing types of the union. See the GraphQL documentation to learn more about [interface](https://graphql.org/learn/schema/#interfaces) and [union](https://graphql.org/learn/schema/#union-types) types.

## Interface Types

Interface types are supported in neo4j-graphql.js through the use of multiple labels in Neo4j. For example, consider the following GraphQL type definitions:

```GraphQL
interface Person {
  id: ID!
  name: String
}

type User implements Person {
  id: ID!
  name: String
  screenName: String
  reviews: [Review] @relation(name: "WROTE", direction: OUT)
}

type Actor implements Person {
  id: ID!
  name: String
  movies: [Movie] @relation(name: "ACTED_IN", direction: OUT)
}

type Movie {
  movieId: ID!
  title: String
}

type Review {
  rating: Int
  created: DateTime
  movie: Movie @relation(name: "REVIEWS", direction: OUT)
}
```

The above GraphQL type definitions would define the following property graph model using neo4j-graphql.js:

![Property graph model](/docs/assets/img/interface-model.png)

<!--
<ul class="graph-diagram-markup" data-internal-scale="1.41" data-external-scale="1">
  <li class="node" data-node-id="0" data-x="0" data-y="0">
    <span class="caption">:Review</span><dl class="properties"><dt>rating</dt><dd>Int</dd><dt>created</dt><dd>DateTime</dd></dl></li>
  <li class="node" data-node-id="1" data-x="-652.1715698242188" data-y="-67.44053077697754">
    <span class="caption">:Person:User</span><dl class="properties"><dt>id</dt><dd>ID!</dd><dt>name</dt><dd>String</dd><dt>screenName</dt><dd>String</dd></dl></li>
  <li class="node" data-node-id="2" data-x="-350.590405836173" data-y="-450.8161956570672">
    <span class="caption">:Person:Actor</span><dl class="properties"><dt>id</dt><dd>ID!</dd><dt>name</dt><dd>String</dd></dl></li>
  <li class="node" data-node-id="3" data-x="203.93539428710938" data-y="-329.8392028808594">
    <span class="caption">:Movie</span><dl class="properties"><dt>movieId</dt><dd>ID!</dd><dt>title</dt><dd>String</dd></dl></li>
  <li class="relationship" data-from="1" data-to="0">
    <span class="type">WROTE</span>
  </li>
  <li class="relationship" data-from="0" data-to="3">
    <span class="type">REVIEWS</span>
  </li>
  <li class="relationship" data-from="2" data-to="3">
    <span class="type">ACTED_IN</span>
  </li>
</ul>
-->

Note that the label `Person` (which represents the interface type) is added to each node of a type implementing the `Person` interface (`User` and `Actor`),

### Interface Mutations

When an interface type is included in the GraphQL type definitions, the generated create mutations will add the additional label for the interface type to any nodes of an implementing type when creating data. For example consider the following mutations.

```GraphQL
mutation {
  u1: CreateUser(name: "Bob", screenName: "bobbyTables", id: "u1") {
    id
  }
  a1: CreateActor(name: "Brad Pitt", id: "a1") {
    id
  }
  m1: CreateMovie(title: "River Runs Through It, A", movieId: "m1") {
    movieId
  }
  am1: AddActorMovies(from: { id: "a1" }, to: { movieId: "m1" }) {
    from {
      id
    }
  }
}
```

This creates the following graph in Neo4j (note the use of multiple labels):

![Simple movie graph with interface types](/docs/assets/img/interface-data.png)

<!--
<ul class="graph-diagram-markup" data-internal-scale="1.41" data-external-scale="1">
  <li class="node" data-node-id="1" data-x="-58.7245244235856" data-y="-3.9691390433209954">
    <span class="caption">:Person:User</span><dl class="properties"><dt>id</dt><dd>"u1"</dd><dt>name</dt><dd>"Bob"</dd><dt>screenName</dt><dd>"bobbyTables"</dd></dl></li>
  <li class="node" data-node-id="2" data-x="-396.65084622430464" data-y="-318.41422244673925">
    <span class="caption">:Person:Actor</span><dl class="properties"><dt>id</dt><dd>"a1"</dd><dt>name</dt><dd>"Brad Pitt"</dd></dl></li>
  <li class="node" data-node-id="3" data-x="85.11565837454289" data-y="-318.41422244673925">
    <span class="caption">:Movie</span><dl class="properties"><dt>movieId</dt><dd>"m1"</dd><dt>title</dt><dd>"River Runs Through It, A"</dd></dl></li>
  <li class="relationship" data-from="2" data-to="3">
    <span class="type">ACTED_IN</span>
  </li>
</ul>
-->

### Interface Queries

A query field is addd to the generated `Query` type for each interface. For example, querying using our `Person` interface.

```GraphQL
query {
  Person {
		name
  }
}
```


```JSON
{
  "data": {
    "Person": [
      {
        "name": "Bob"
      },
      {
        "name": "Brad Pitt"
      }
    ]
  }
}
```

The `__typename` introspection field can be added to the selection set to determine the concrete type of the object. 

```GraphQL
query {
  Person {
		name
    __typename
  }
}
```

```JSON
{
  "data": {
    "Person": [
      {
        "name": "Bob",
        "__typename": "User"
      },
      {
        "name": "Brad Pitt",
        "__typename": "Actor"
      }
    ]
  }
}
```

[Inline fragments](https://graphql.org/learn/queries/#inline-fragments) can be used to access fields of the concrete types in the selection set.

```GraphQL
query {
  Person {
    name
    __typename
    ... on Actor {
      movies {
        title
      }
    }

    ... on User {
      screenName
    }
  }
}
```


```JSON
{
  "data": {
    "Person": [
      {
        "name": "Bob",
        "__typename": "User",
        "screenName": "bobbyTables"
      },
      {
        "name": "Brad Pitt",
        "__typename": "Actor",
        "movies": [
          {
            "title": "River Runs Through It, A"
          }
        ]
      }
    ]
  }
}
```

The generated filter arguments can be used for interface types. Note however that only fields in the interface definition are included in the generated filter arguments as those apply to all concrete types.

```GraphQL
query {
  Person(filter: {name_contains:"Brad"}) {
		name
    __typename
    ... on Actor {
      movies {
        title
      }
    }
    
    ... on User {
      screenName
    }
  }
}
```

```JSON
{
  "data": {
    "Person": [
      {
        "name": "Brad Pitt",
        "__typename": "Actor",
        "movies": [
          {
            "title": "River Runs Through It, A"
          }
        ]
      }
    ]
  }
}
```

We can also use intefaces when defining relationship fields. For example:

```GraphQL
  friends: [Person] @relation(name: "FRIEND_OF", direction: OUT)

```

## Union Types

// Find some other schema that isn't built off the previous

### Defining In SDL

### Union Mutations

### Union Queries


Use with @cypher directive field for full text index

### Use Without Specifying Relationship Type

## Using Union and Interface Types Together

Note that we cannot have interfaces in unions, but we can include the _concrete_ type in a union.

> NOTE: not yet implmented for relationship types