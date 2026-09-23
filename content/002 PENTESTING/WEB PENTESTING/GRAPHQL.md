---
Primary_category: "[[WEB TECHNOLOGIES]]"
title: "GRAPHQL"
draft: false
banner: "https://images.unsplash.com/photo-1589763472885-46dd5b282f52?q=80&w=1748&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D"
banner_y: 0.88286
tags:
cssclasses:
---

###### PRIMARY CATEGORY → [[WEB TECHNOLOGIES]]

#### *Theory*

***GraphQL*** is a query language and runtime for *APIs* that allows clients to request exactly the data they need from a server.

Unlike traditional *REST APIs*, where multiple endpoints typically expose different resources, *GraphQL* commonly uses a single endpoint and lets the client define the structure of the requested data

```bash
<TARGET>/graphql
<TARGET>/api/graphql
```

*GraphQL APIs* are defined through a schema containing types, fields, queries and mutations, which the server resolves when processing client requests

Misconfigurations or insecure implementations can expose sensitive information, authorization flaws, excesive data access or other attack vectors

---

#### *Enumeration*

##### *Identifying the GraphQL Engine*

First, we can proceed as follows to identify which *GraphQL* engine is being used

- ***[graphw00f](https://github.com/dolevf/graphw00f)***

***Setup***

```bash
git clone https://github.com/dolevf/graphw00f graphw00f
cd !$
```

***Usage***

```bash
python3 main.py --fingerprint --detect --target '<TARGET>'
```

Then, simply check the resource below, which provides more in-depth information about the identified *GraphQL* engine

> ***[GraphQL-Threat-Matrix](https://github.com/nicholasaleks/graphql-threat-matrix)***

> [!IMPORTANT]
>
> ***We should always check whether the backend is using a [graphiql](https://github.com/graphql/graphiql) interface under its `/graphql` endpoint***
>

##### *Introspection*

*Introspection* is a *GraphQL* feature that enables users to query *GraphQL API* about the structure of the backend system

Therefore, a user could use an introspection query to obtain all queries supported by the *API* schema

> ***Introspection comes enabled by default on certain GraphQL engines, such as graphene***

![[GRAPHQL-20260920170329247.webp|450]]

> ***Zoom in***

###### *Types*

> ***All GraphQL Types supported by the backend server***

> [!DANGER]- *Query*
>
> ```graphql
> { 
>   __schema { 
>     types { 
>       name 
>     } 
>   } 
> } 
> ```
>

###### *All fields for a specific Type*

- ***Standard Type → OBJECT***

> [!DANGER]- *Query*
>
> ```graphql
> {
>   __type(name: "<TYPE>") {
>     name
>     fields {
>       name
>       type {
>         name
>         kind
>       }
>     }
>   }
> }
> ```
>

- ***Input Type → INPUT_OBJECT***

> [!DANGER]- *Query*
>
> ```graphql
> {
>   __type(name: "<TYPE>") {
>     name
>     kind
>     inputFields {
>       name
>     }
>   }
> }
> ```
>

###### *Queries*

> ***All GraphQL Queries supported by the backend server***

> [!DANGER]- *Query*
>
> ```graphql
> {
>   __schema {
>     queryType {
>       fields {
>         name
>         description
>       }
>     }
>   }
> }
> ```
>

After enumerating them, we can call one of them as follows

> ***allEmployees Query***

> [!DANGER]- *e.g.*
>
> ```graphql
> {
>   allEmployees {
>     id
>     username
>     role
>   }
> }
> ```
>

###### *Mutations*

> ***All GraphQL Mutations supported by the backend server***

> [!DANGER]- *Query*
>
> ```graphql
> query {
>   __schema {
>     mutationType {
>       name
>       fields {
>         name
>         args {
>           name
>           defaultValue
>           type {
>             ...TypeRef
>           }
>         }
>       }
>     }
>   }
> }
> 
> fragment TypeRef on __Type {
>   kind
>   name
>   ofType {
>     kind
>     name
>     ofType {
>       kind
>       name
>       ofType {
>         kind
>         name
>         ofType {
>           kind
>           name
>           ofType {
>             kind
>             name
>             ofType {
>               kind
>               name
>               ofType {
>                 kind
>                 name
>               }
>             }
>           }
>         }
>       }
>     }
>   }
> }
> ```
>

After enumerating them, we can carry out a mutation as follows

- ***Standard Mutation***

> ***registerUser Mutation***

> [!DANGER]- *e.g.*
>
> ```graphql
> mutation {
>   registerUser(input: {username: "<USER>", password: "<PASSWD>", role: "<ROLE>", msg: "<MSG>"}) {
>     user {
>       username
>       password
>       msg
>       role
>     }
>   }
> }
> ```
>

###### *All information*

> ***Types, fields and queries supported by the backend***

> [!DANGER]- *Query*
>
> ```graphql
> query IntrospectionQuery {
>       __schema {
>         queryType { name }
>         mutationType { name }
>         subscriptionType { name }
>         types {
>           ...FullType
>         }
>         directives {
>           name
>           description
>           
>           locations
>           args {
>             ...InputValue
>           }
>         }
>       }
>     }
> 
>     fragment FullType on __Type {
>       kind
>       name
>       description
>       
>       fields(includeDeprecated: true) {
>         name
>         description
>         args {
>           ...InputValue
>         }
>         type {
>           ...TypeRef
>         }
>         isDeprecated
>         deprecationReason
>       }
>       inputFields {
>         ...InputValue
>       }
>       interfaces {
>         ...TypeRef
>       }
>       enumValues(includeDeprecated: true) {
>         name
>         description
>         isDeprecated
>         deprecationReason
>       }
>       possibleTypes {
>         ...TypeRef
>       }
>     }
> 
>     fragment InputValue on __InputValue {
>       name
>       description
>       type { ...TypeRef }
>       defaultValue
>     }
> 
>     fragment TypeRef on __Type {
>       kind
>       name
>       ofType {
>         kind
>         name
>         ofType {
>           kind
>           name
>           ofType {
>             kind
>             name
>             ofType {
>               kind
>               name
>               ofType {
>                 kind
>                 name
>                 ofType {
>                   kind
>                   name
>                   ofType {
>                     kind
>                     name
>                   }
>                 }
>               }
>             }
>           }
>         }
>       }
>     }
> ```
>

Since the result of this query is quite large and complex, we can use ***[GraphQL-Voyager](https://github.com/APIs-guru/graphql-voyager)*** to visualize the schema as an interactive graph

> ***e.g.***

![[GRAPHQL-20260920172005521.webp|450]]

> ***Zoom in***

To do so, simply access ***[here](https://apis.guru/graphql-voyager/)***, select *CHANGE SCHEMA*, go to *INTROSPECTION* section and paste the output of previous query

![[GRAPHQL-20260920184623833.webp|450]]

> ***Zoom in***

##### *Auditing a GraphQL Endpoint*

> ***Automated***

- ***[GraphQL-Cop](https://github.com/dolevf/graphql-cop)***

***Setup***

```bash
git clone https://github.com/dolevf/graphql-cop GraphQL-Cop
cd !$ && python3 -m venv .venv
. !$/bin/activate && pip3 install -r requirements.txt
```

***Usage***

```bash
python3 graphql-cop.py -t '<TARGET>'
```

> [!DANGER]- *e.g.*
>
> ```bash
> python3 graphql-cop.py -t 'http://10.10.10.5:8000/graphql'
> ```
>

---

#### *Insecure Direct Object Reference*

> ***[[IDOR]]***

Let's suppose that we face a web application that carries out the query below to a *GraphQL* endpoint to retrieve information about the current logged-in user

![[GRAPHQL-20260920180128117.webp|450]]

> ***Zoom in***

The web app passes our username to the **`username`** parameter of the **`user`** field

Therefore, if we know valid usernames, we could try to request information about any of them, which would be an authorization flaw

![[GRAPHQL-20260920181556930.webp|450]]

> ***Zoom in***

However, the information we get is not sensitive at all. So, we could send an *introspection* query to list all existing fields for the *userObject* type

```graphql
{ __type( name: \"UserObject\" ) { name fields { name type { name kind } } } }
```

![[GRAPHQL-20260920182149922.webp|450]]

> ***Zoom in***

And we have a **`password`** field

So, we can leverage the lack of authorization to retrieve this sensitive information for a specific user

![[GRAPHQL-20260920182507385.webp|450]]

> ***Zoom in***

---

#### *SQL Injection*

> ***[[SQLi]]***

Since *GrapQL* is a query language, it has to fetch data from some kind of storage, typically a database

So, as with any other web application, we could test for *SQL Injection* to see whether the given *GraphQL API* properly sanitizes the user input

Therefore, we should look for any *GraphQL* that requires or supports arguments, and analyze these arguments for potential *SQL Injection*

So, following the ***[[#Insecure Direct Object Reference|IDOR]]*** example, we have the following *GraphQL* query used by the web application in question to retrieve some information for a specific user

![[GRAPHQL-20260920185618890.webp|450]]

> ***Zoom in***

The given user is specified in the **`username`** parameter of the **`user`** field

With this in mind, we can test it for *SQL Injection*. If we enter a simple quote, we get the following error, which indicates that it's vulnerable

![[GRAPHQL-20260920185911731.webp|450]]

> ***Zoom in***

Since the error is displayed in the response, it seems that we can proceed with an ***[[UNION BASED SQLI|UNION-based]]*** approach to extract sensitive information

![[GRAPHQL-20260920190129902.webp|450]]

> ***Zoom in***