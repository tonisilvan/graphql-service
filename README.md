
# Full‑Stack GraphQL — React (Next.js) + Node (Apollo Server) + Postgres

A pragmatic **full‑stack GraphQL** starter that shows **mutations, queries, pagination, auth and optimistic UI** from a **React (Next.js + Apollo Client)** frontend talking to a **Node.js (Apollo Server)** backend. Uses **Postgres + Prisma** for data access, **JWT** for auth, and ships **Docker** and **docker‑compose** for one‑command local setup. Minimal but production‑tilted.

> Goal: ready‑to‑review proof that I can design a modern GraphQL client/server, including auth flows, caching, optimistic updates, and containerization.

---

## ✨ Features

- **GraphQL API** with Apollo Server (Node 20+)
- **Next.js** (App Router) + **Apollo Client** with normalized cache
- **JWT Auth** (login/register/refresh) + `@auth` directive on the server
- **CRUD** for a simple domain: **Products**, **Customers**, **Orders**
- **Pagination** (cursor) and **filtering/sorting**
- **Optimistic UI** for create/update/delete
- **Form validation** (Zod) and type‑safe GraphQL (Codegen)
- **Prisma ORM** with Postgres + migrations
- **Docker** & **docker‑compose** for DB and server
- **Testing**: Vitest (client) + Jest (server)
- **ESLint/Prettier** configured
- Optional **subscriptions** via WebSocket

---

## 🏗 Project Structure

```
fullstack-graphql/
├─ apps/
│  ├─ web/                 # Next.js 14 (React 18) + Apollo Client
│  └─ api/                 # Node (Apollo Server) + Prisma + JWT
├─ packages/
│  ├─ schema/              # GraphQL SDL + fragments + codegen config
│  └─ tsconfig/            # TS base configs
├─ docker-compose.yml
└─ README.md
```

- Monorepo layout (pnpm workspaces). You can also run each app standalone.

---

## 🧬 GraphQL Schema (excerpt)

```graphql
directive @auth(roles: [Role!]) on FIELD_DEFINITION

enum Role { USER ADMIN }

type Query {
  me: User @auth
  products(after: String, first: Int = 20, filter: ProductFilter, sort: [ProductSort!]): ProductConnection!
  orders(after: String, first: Int = 20): OrderConnection! @auth
  product(id: ID!): Product
}

type Mutation {
  register(input: RegisterInput!): AuthPayload!
  login(email: String!, password: String!): AuthPayload!
  refreshToken(token: String!): AuthPayload!

  createProduct(input: ProductInput!): Product! @auth(roles:[ADMIN])
  updateProduct(id: ID!, input: ProductInput!): Product! @auth(roles:[ADMIN])
  deleteProduct(id: ID!): Boolean! @auth(roles:[ADMIN])

  placeOrder(input: PlaceOrderInput!): Order! @auth
}

type Subscription {
  orderCreated: Order! @auth
}
```

Relay‑style connections (cursor pagination) for lists.

---

## 🔐 Auth Flow

- `register` ➜ returns `accessToken` (short TTL) + `refreshToken` (long TTL)
- Client stores **accessToken in memory** and **refreshToken in httpOnly cookie**
- Apollo link auto‑attaches `Authorization: Bearer <accessToken>`
- Silent refresh via `/refresh` mutation on 401/expiry
- Role gates with `@auth` directive

---

## 🚀 Quick Start

### Prerequisites
- Node 20+ and pnpm 9+
- Docker Desktop
- (Optional) OpenSSL for generating JWT keys

### 1) Clone & install
```bash
pnpm install
```

### 2) Env files
Create `.env` files (samples below).

**apps/api/.env**
```
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fullstack
JWT_SECRET=dev-secret-change-me
PORT=4000
```

**apps/web/.env.local**
```
NEXT_PUBLIC_GRAPHQL_URL=http://localhost:4000/graphql
```

### 3) Start Postgres (Docker) + run migrations
```bash
docker compose up -d
pnpm -C apps/api prisma:migrate   # or: pnpm -C apps/api prisma generate && prisma migrate dev
pnpm -C apps/api seed
```

### 4) Run dev servers
```bash
# API
pnpm -C apps/api dev

# Web
pnpm -C apps/web dev
```

- API runs at `http://localhost:4000/graphql` (GraphQL Playground enabled in dev)
- Web runs at `http://localhost:3000`

---

## 🧩 Frontend (Next.js + Apollo Client)

Key pieces:
- `ApolloProvider` with an `HttpLink` + `authLink` to inject JWT
- `refreshLink` to intercept 401 and execute `refreshToken`
- Generated hooks (via GraphQL Codegen) like `useProductsQuery`, `useCreateProductMutation`

**Example page snippet:**

```tsx
// apps/web/app/products/page.tsx
"use client";
import { useProductsQuery, useCreateProductMutation } from "@schema/generated";
import { useState } from "react";

export default function ProductsPage() {
  const { data, loading, fetchMore } = useProductsQuery({ variables: { first: 20 } });
  const [createProduct] = useCreateProductMutation();
  const [name, setName] = useState("");

  const onAdd = async () => {
    await createProduct({
      variables: { input: { name, sku: crypto.randomUUID(), price: 19.99, stock: 10 } },
      optimisticResponse: {
        createProduct: { __typename: "Product", id: "temp", name, sku: "TEMP", price: 19.99, stock: 10 }
      },
      update: (cache, { data }) => {
        // merge new product into ProductConnection...
      }
    });
    setName("");
  };

  if (loading) return <p>Loading...</p>;
  return (
    <div>
      <h1>Products</h1>
      <input value={name} onChange={e => setName(e.target.value)} placeholder="Name" />
      <button onClick={onAdd}>Add</button>
      <ul>{data?.products.edges.map(e => <li key={e.node.id}>{e.node.name}</li>)}</ul>
      <button onClick={() => fetchMore({ variables: { after: data?.products.pageInfo.endCursor } })}>
        Load more
      </button>
    </div>
  );
}
```

---

## 🛠 Backend (Apollo Server + Prisma)

- Context builds user from `Authorization` header
- `@auth` directive checks roles (schema transforms)
- Prisma models: `User`, `Product`, `Order`, `OrderItem`
- Resolvers split by domain; dataloaders for N+1 avoidance

**Sample resolver (excerpt):**
```ts
// apps/api/src/resolvers/mutation.ts
export const Mutation = {
  login: async (_: any, { email, password }: any, { services }) => {
    const user = await services.auth.verify(email, password);
    return services.auth.issueTokens(user);
  },
  createProduct: async (_: any, { input }: any, { prisma, user }) => {
    // auth checked by directive, extra safeguard here if needed
    return prisma.product.create({ data: input });
  },
};
```

---

## 🧪 Testing

- **Server**: Jest + supertest (graphql endpoint), prisma test DB
- **Client**: Vitest + React Testing Library + MockedProvider

---

## 🐳 Docker Compose

Bring up DB + API in one go:

```yaml
# docker-compose.yml (excerpt)
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: fullstack
    ports: [ "5432:5432" ]
  api:
    build: ./apps/api
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/fullstack
      JWT_SECRET: dev-secret-change-me
      PORT: 4000
    ports: [ "4000:4000" ]
    depends_on: [ db ]
```

Add `web` service later if you want containerized frontend.

---

## 🧭 Roadmap (short)

- [ ] Add `web` to docker‑compose
- [ ] Subscriptions (`orderCreated`) end‑to‑end
- [ ] Refresh token rotation + revoke
- [ ] E2E tests (Playwright) happy paths
- [ ] GitHub Actions CI (build + tests + docker)
- [ ] Minimal k8s manifests (api + db)

---

## 📝 Scripts (pnpm)

```bash
pnpm -C apps/api dev          # ts-node-dev
pnpm -C apps/api prisma:migrate
pnpm -C apps/web dev          # next dev
pnpm -C packages/schema codegen
```

---

## 📌 Why this is a good interview project

- Shows **client/server GraphQL** with real features recruiters look for
- Demonstrates **auth + cache + optimistic UI**
- Uses **typed tooling** (Prisma, Codegen) and **containers**
- Easy to run locally with **docker‑compose**
