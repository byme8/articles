
In this article I want to showcase new way to create web apis with GraphQL. We will have a look on what benifits it has comparing to traditional REST API, when it is better to use it and how we can transform existing REST API to support GraphQL.

# Over and under fetching

First of all, I guess we need to figure out issues can be solved with. The main two issues would be "under feathing" and "over fetching". They are common when we are working with REST APIs. Under-fetching occurs when an initial API request doesn't supply enough data, forcing the application to make additional requests. Over-fetching, on the other hand, happens when an API request returns more data than the client needs.

Consider a scenario where a developer works on a user profile that needs to load user personal information (e.g., name, email, and profile picture) and user roles (e.g., admin, subscriber, etc.). In a traditional RESTful approach, fetching this data could require at least two different endpoints: one for personal information and another for roles.

If you fetch from the personal information endpoint and it doesn't include the user roles, you've under-fetched and must make an additional request to the roles endpoint. Conversely, if the personal information endpoint returns all available data about a user—including details not needed for the current view—you've over-fetched, wasting bandwidth and potentially slowing down the application.

The GraphQL helps to solve this issue rather easy. Instead of having multiple endpoints, there's a single endpoint that accepts queries. These queries define exactly what data the client needs, eliminating under-fetching. The client can request different data types in a single request.

Let's look at our user profile example. Instead of making separate requests for personal information and roles, we can make a single GraphQL query:
``` graphql
query {
  user(id: 1) {
    name
    email
    profilePicture
    roles {
      title
    }
  }
}
```

This query asks for specific fields (name, email, profilePicture, and roles) for the user with an id of 1. The server will return a JSON object that mirrors the shape of the query and includes just the requested fields.

The flexibility of GraphQL queries also solves the problem of over-fetching. Because the client specifies exactly what data it needs, the server will never send unnecessary data. Using our example, if we don't need the user's profile picture for a certain view, we can simply leave it out of the query:
``` graphql
query {
  user(id: 1) {
    name
    email
    roles {
      title
    }
  }
}
```

In this query, the server will not send the profile picture, even if it's available. 

# Key differences with REST

## Query

GraphQL's strict protocol is an essential part of its power and is a key distinction from REST APIs, where conventions can vary significantly. Nowdays, all major tech companies has its own REST best practices that sometimes can be completly uncompatible.

At the same time, GraphQL APIs are organized in terms of types and fields, not endpoints. GraphQL uses a strong type system to describe the capabilities of an API, and all the types of data it can return, enabling developer tools and ensuring the APIs behave in predictable ways.

Here example how the schema may look:
``` graphql
type Query {
  users(page: Int!, size: Int!): [User!]!
  user(id: Int!): User
}

type Mutation {
  createUser(name: String!, email: String!): User!
}

type Role {
  id: Int!
  name: String!
}

type User {
  id: Int!
  name: String!
  email: String!
  roles: [Role!]!
}
```

## Versioning

Versioning is an essential aspect of maintaining APIs, especially when the API undergoes significant changes. In traditional REST APIs, versioning often involves either changing the URL of an endpoint or including a version number in the request header. This approach has the downside of potentially breaking the application for any clients not updated to use the new version of the API.However, GraphQL takes a fundamentally different approach to versioning, helping avoid the issues associated with traditional versioning methods. 

Instead of creating new versions of the entire API, developers can simply extend the existing GraphQL schema with new fields to support new functionality. Clients that haven't been updated to understand these new fields will just ignore them, and the API remains backward compatible.

In case when we need to remove a field, GraphQL supports deprication, which signals to developers that a field is no longer in use and will be removed in the future. Even after a field is deprecated, it still works, so clients using that field will not break immediately. This provides time for clients to transition away from the deprecated field.

The strong type system in GraphQL helps to ensure that changes are compatible with existing queries. It prevents incompatible changes like altering the type of a field.

In essence, GraphQL's approach to versioning avoids the need for version numbers and breaking changes. It's more flexible and forgiving, and it lets developers add new features to an API without disrupting existing users. As a result, API evolution becomes a smoother process, fostering a better developer experience and ensuring more reliable applications.

## Client generation

In GraphQL, client generation leverages the API schema and introspection feature. Given that the GraphQL schema is strongly typed and the types and relationships are well defined, it's possible to use this information to automatically generate client code. The client can be in various languages like TypeScript, C#, etc. It reduces the amount of boilerplate code that developers need to write, saving time and minimizing the risk of manual coding errors. 

Client generation works harmoniously with GraphQL's field deprecation semantic. When a field is deprecated, it can still be included in the schema and thus in the generated client code, but it's marked as deprecated.

Developers can see the deprecation warning directly in their IDE while they're coding. This helps them move away from using deprecated fields in a smooth and controlled manner. Over time, as developers stop using the deprecated field, it can be safely removed from the schema without breaking any clients.


# Converting existing REST API into GraphQL API

## Initial setup

Now lets have all look on how we can convert existing REST API into GraphQL API. 
The initial endpooints look like that:
``` csharp
app.UseEndpoints(
    o =>
    {
        o.MapSwagger();
        o.MapGet("/users", 
            async (int page, int size, UserService service) => await service.GetUsers(page, size));
        
        o.MapGet("/users/{id}",
            async (int id, UserService service) => await service.GetUser(id));
        
        o.MapGet("/users/{id}/roles",
            async (int id, RoleService service) => await service.GetRolesByUserId(new[] { id }));
        
        o.MapPost("/users", 
            async (UserCreationRequest request, UserService service) => await service.CreateUser(request.Name, request.Name));
    });
```

> The article contains only selected source code to be short.
> If you want to see the full source code go to demo repository https://github.com/byme8/DemoGraphQL
> There are two branches:
> - master - contains intial REST ApI
> - graphql - contains final GraphQL API

A basic endpoints that handle page from the under and over featching example. First thing you need to do is to install `` HotChocolate.AspNetCore `` Nuget package. 
``` bash
dotnet add package HotChocolate.AspNetCore
```

The `` HotChocolate `` is a set of tools that allows us to recieve and process GraphQL queries. It has a nice integration with the AspNet Core pipeline. The full documentation can be find [here](https://chillicream.com/docs/hotchocolate/v13).

Now bootstrap GraphQL server:
``` csharp
+ services
+    .AddGraphQLServer()
+    .AddQueryType<Query>();

// ...

app.UseEndpoints(
    o =>
    {
+        o.MapGraphQL();
        o.MapSwagger();
        o.MapGet("/users", 
            async (int page, int size, UserService service) => await service.GetUsers(page, size));
// ...
public class Query 
{
  public DateTimeOffset Utc() => DateTimeOffset.UtcNow;
}  
```
When we launch our application, we can access GraphQL endpoint `` http://localhost:1234/graphql ``. If if open via browser we will see a GraphQL IDE. Here we can check the GraphQL schema and execute requests.

![GraphQL Web IDE](./graphql-ide.png "GraphQL Web IDE")

The initial setup is ready and we can start to transform our REST APIs.

## Queries

Let's transform the next endpoint first:
``` csharp
        o.MapGet("/users/{id}",
            async (int id, UserService service) => await service.GetUser(id));
```

To do it we can just create a new method in our `` Query `` class:
``` csharp
public class Query 
{
  public DateTimeOffset Utc() => DateTimeOffset.UtcNow;

+  public Task<User?> GetUser(int id, [Service] UserService userService)
+    => userService.GetUser(id);
}  
```
Done! If we open our GraphQL IDE and check GraphQL schema, it would look like that:
``` graphql
type Query {
  utc: DateTime!
  user(id: Int!): User
}

type Role {
  id: Int!
  name: String!
}

type User {
  id: Int!
  name: String!
  email: String!
  roles: [Role!]!
}

"""
The `DateTime` scalar represents an ISO-8601 compliant date time type.
"""
scalar DateTime
```

The `` HotChocolate `` automaticly detects what types are exposed and add them to the schema.

Let's move one and transform another endpoint. However, do it a bit differently. We still can add it to the `` Query `` class, but it is not a great idea to place everything into one class. We can split it to look very close to what we have when we using controllers from `` AspNet Core ``.

To make it work create new class `` UserQueryExtensions ``. It will look like this:
``` csharp
 services
    .AddGraphQLServer()
    .AddQueryType<Query>()
+    .AddTypeExtension<UserQueryExtensions>();

// ..

+ [ExtendObjectType(typeof(Query))]
+ public class UserQueryExtensions
+ {
+     public Task<IEnumerable<User>> GetUsers(int page, int size, [Service] UserService userService)
+         => userService.GetUsers(page, size);
+     
+     public Task<User?> GetUser(int id, [Service] UserService userService)
+         => userService.GetUser(id);
+ }
```

Here we created the class extends another GraphQL type. In our case, it is a root type `` Query ``.