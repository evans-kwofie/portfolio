---
title: Type-Safe GraphQL Without Exposing Your Schema
description: How to set up graphql-request and GraphQL Code Generator in a BFF pattern so you get full type safety without leaking your schema to the browser.
publishDate: December 17, 2025
tags: ["software development", "graphql", "typescript"]
draft: true
---

In my previous post, [Why I'm No Longer a GraphQL Maximalist](/posts/graphql-frontend-learnings/), I talked about the security trade-offs of GraphQL—how your queries are visible in the network tab and how introspection can expose your entire schema. But here's the thing: you don't have to give up GraphQL's developer experience to solve these problems.

The trick is to keep GraphQL on the server and expose clean REST-like endpoints to the browser. You still get type-safe queries and code generation during development, but your users never see a single GraphQL query.

Here's how to set it up with **graphql-request** and **GraphQL Code Generator**.

### Install the dependencies

```bash
npm install graphql graphql-request
npm install -D @graphql-codegen/cli @graphql-codegen/client-preset
```

### Configure GraphQL Code Generator

Create a `codegen.ts` file at the root of your project:

```typescript
// codegen.ts
import type { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  schema: process.env.GRAPHQL_ENDPOINT,
  documents: ["src/**/*.graphql"],
  generates: {
    "./src/gql/": {
      preset: "client",
      config: {
        documentMode: "string",
      },
    },
  },
};

export default config;
```

Add the codegen script to your `package.json`:

```json
{
  "scripts": {
    "codegen": "graphql-codegen --config codegen.ts"
  }
}
```

The key here is that the schema is only fetched at **build time**. Your GraphQL endpoint and its introspection are never accessed from the browser.

### Write your queries

Create `.graphql` files for your queries. I like to keep them next to the features that use them:

```graphql
# src/features/users/queries.graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    first_name
    last_name
    email
    avatar_url
  }
}

query GetUserPosts($userId: ID!, $limit: Int!) {
  user(id: $userId) {
    posts(limit: $limit) {
      id
      title
      excerpt
      created_at
    }
  }
}
```

Run `npm run codegen` and you'll get fully typed document exports in `src/gql/`.

### Set up the GraphQL client

This client should **only** be imported in server-side code:

```typescript
// src/lib/graphql-client.ts
import { GraphQLClient } from "graphql-request";

export const graphqlClient = new GraphQLClient(
  process.env.GRAPHQL_ENDPOINT!,
  {
    headers: {
      "x-hasura-admin-secret": process.env.HASURA_ADMIN_SECRET!,
    },
  }
);
```

If you're using Hasura, that's your admin secret. For other backends, it might be an API key or a service token. The point is: this secret lives in environment variables on your server and never touches the browser.

### Next.js App Router example

```typescript
// src/app/api/users/[id]/route.ts
import { graphqlClient } from "@/lib/graphql-client";
import { GetUserDocument } from "@/gql/graphql";

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  try {
    const data = await graphqlClient.request(GetUserDocument, {
      id: params.id,
    });

    return Response.json({
      id: data.user.id,
      name: `${data.user.first_name} ${data.user.last_name}`,
      email: data.user.email,
      avatarUrl: data.user.avatar_url,
    });
  } catch (error) {
    return Response.json({ error: "User not found" }, { status: 404 });
  }
}
```

Your frontend just calls `/api/users/123` and gets back clean JSON. No GraphQL in sight.

### Remix loader example

```typescript
// src/routes/users.$id.tsx
import { json, type LoaderFunctionArgs } from "@remix-run/node";
import { useLoaderData } from "@remix-run/react";
import { graphqlClient } from "@/lib/graphql-client";
import { GetUserDocument, GetUserPostsDocument } from "@/gql/graphql";

export async function loader({ params }: LoaderFunctionArgs) {
  const [userData, postsData] = await Promise.all([
    graphqlClient.request(GetUserDocument, { id: params.id }),
    graphqlClient.request(GetUserPostsDocument, {
      userId: params.id,
      limit: 5
    }),
  ]);

  return json({
    user: {
      id: userData.user.id,
      name: `${userData.user.first_name} ${userData.user.last_name}`,
      email: userData.user.email,
    },
    posts: postsData.user.posts.map((post) => ({
      id: post.id,
      title: post.title,
      excerpt: post.excerpt,
    })),
  });
}

export default function UserProfile() {
  const { user, posts } = useLoaderData<typeof loader>();

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>

      <h2>Recent Posts</h2>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### TanStack Start example

```typescript
// src/routes/users/$id.tsx
import { createFileRoute } from "@tanstack/react-router";
import { createServerFn } from "@tanstack/start";
import { graphqlClient } from "@/lib/graphql-client";
import { GetUserDocument } from "@/gql/graphql";

const getUser = createServerFn("GET", async (id: string) => {
  const data = await graphqlClient.request(GetUserDocument, { id });

  return {
    id: data.user.id,
    name: `${data.user.first_name} ${data.user.last_name}`,
    email: data.user.email,
  };
});

export const Route = createFileRoute("/users/$id")({
  loader: ({ params }) => getUser(params.id),
  component: UserProfile,
});

function UserProfile() {
  const user = Route.useLoaderData();

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### What the browser sees

With this pattern, if someone opens the network tab, they'll see:

```
GET /api/users/123
Response: { "id": "123", "name": "John Doe", "email": "john@example.com" }
```

No GraphQL queries. No schema hints. No table names. Just clean JSON from a REST-like endpoint.

### The tradeoff

You lose the ability to do client-side GraphQL operations, which means no Apollo DevTools and no client-side caching based on GraphQL types. But honestly, if you're using a framework with built-in data loading (Next.js, Remix, TanStack Start), you probably don't need those anyway. The framework handles caching and the server handles data fetching.

You keep all the parts of GraphQL that actually matter for developer productivity: the type generation, the ability to fetch exactly what you need in a single query, and the schema as a contract between your frontend and backend teams.
