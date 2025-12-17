---
title: Why I'm No Longer a GraphQL Maximalist
description: GraphQL feels like magic until you hit production. Here are the hard lessons I learned about security, introspection, and knowing when to just use REST.
publishDate: December 17, 2025
tags: ["software development", "graphql"]
draft: false
---

Three years ago, I joined a team whose tech stack was built almost entirely around GraphQL. We weren't building custom backends from scratch; we were using engines like **Hasura**. At first, it felt like a superpower. Being able to explore the schema and get instant type-safety with a bit of **GraphQL Code Generator** made development feel incredibly fast.

I eventually learned to set up these backends myself, and for a long time, it was smooth sailing. But after two and a half years of maintaining these systems in production, I've realized that the "magic" comes with some significant security and architectural trade-offs that don't get talked about enough.

### The Exposure Problem

One of the first things you notice when working with GraphQL is how much it reveals to the outside world. In a traditional REST API, an endpoint like `/api/v1/profile` doesn't tell a user anything about your database structure. In GraphQL, the query is part of the request payload, and it's right there in the browser's **Network Tab** for anyone to see.

When you send a request, you are essentially broadcasting your table names and field structures. If you have a query like this:

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    first_name
    last_name
    email
  }
}
```

Anyone looking at that traffic now knows you have a `user` table with those specific columns. If your permissions or Row Level Security (RLS) aren't perfectly configured, you've basically handed a map of your database to anyone who wants to look for a weakness.

### The Risk of Introspection

By default, GraphQL has a feature called **Introspection**. This is what allows tools like GraphiQL to show you documentation and autocomplete your queries. It's a great developer tool, but it's a massive liability in production. If it's left on, a bad actor can send one query and get a complete list of every single data type, relationship, and mutation available in your system. It turns the process of hacking your API from a guessing game into a guided tour.

### A Better Way: Persisted Queries

If you want the benefits of GraphQL without exposing your entire schema, you should look into **Persisted Queries**.

Instead of the client sending a giant string of GraphQL code over the network, you store your queries on the server beforehand. The client then only sends a unique ID or a hash (like a long string of random characters) to the server. The server sees that ID, looks up the corresponding query in its own database, and runs it.

Here's what a normal GraphQL request looks like in your network tab:

```json
// POST /graphql - What everyone can see
{
  "query": "query GetUser($id: ID!) { user(id: $id) { id first_name last_name email } }",
  "variables": { "id": "123" }
}
```

With persisted queries, the same request becomes:

```json
// POST /graphql - Much harder to reverse-engineer
{
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "abc123def456..."
    }
  },
  "variables": { "id": "123" }
}
```

On the server side, you maintain a map of hashes to queries:

```typescript
// Server-side query map (never exposed to the client)
const persistedQueries = {
  "abc123def456...": `
    query GetUser($id: ID!) {
      user(id: $id) {
        id
        first_name
        last_name
        email
      }
    }
  `,
  "xyz789ghi012...": `
    query GetPosts($limit: Int!) {
      posts(limit: $limit) {
        id
        title
        created_at
      }
    }
  `
};
```

This does two things: it cleans up your network traffic so no one can read your database structure, and it prevents attackers from sending "malicious" custom queries to your backend. It essentially turns your flexible GraphQL API into a set of fixed, secure endpoints.

### Making it Practical

So, how do we actually handle this? First, **always turn off introspection in production.** There is rarely a reason for the public to have a map of your schema.

Second, consider using a **Backend-for-Frontend (BFF)**. If you are using Next.js, Remix, or TanStack Start, you can keep your GraphQL queries on the server side. The browser talks to your server-side "loader" or "action," and that server code talks to GraphQL using a private secret. This keeps the GraphQL logic completely hidden from the user's browser.

### Do You Actually Need It?

The most important lesson I've learned is to stop treating GraphQL as the "new default." Its real strength is managing deeply nested, relational data—think of a social media feed where a post has an author, and that author has comments, and those comments have likes.

If your app is mostly flat data or simple lists, you're likely adding a lot of complexity for no reason. You'll spend your time managing codegen, securing endpoints, and fighting N+1 issues without getting the actual benefits of the technology. For many projects, a traditional REST API is still the better, simpler choice.
