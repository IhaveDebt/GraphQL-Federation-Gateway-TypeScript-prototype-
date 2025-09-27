/**
 * GraphQL Federation Gateway (federation_gateway.ts)
 *
 * Minimal gateway prototype that merges two simple schemas and forwards requests.
 * For a production gateway use Apollo Federation or GraphQL Mesh.
 *
 * Usage:
 *  ts-node src/federation_gateway.ts
 */
import express from 'express';
import fetch from 'node-fetch';
import { buildSchema, graphql } from 'graphql';

// Demo schemas (in reality these are remote services)
const schemaA = buildSchema(`
  type User { id: ID!, name: String! }
  type Query { userById(id: ID!): User }
`);
const rootA = {
  userById: ({ id }: any) => ({ id, name: `User-${id}` })
};

const schemaB = buildSchema(`
  type Post { id: ID!, authorId: ID!, title: String! }
  type Query { postsByAuthor(authorId: ID!): [Post] }
`);
const rootB = {
  postsByAuthor: ({ authorId }: any) => ([{ id: 'p1', authorId, title: 'Hello World' }])
};

// Gateway composes: Query userAndPosts(id) -> fetch from both
const gatewaySchema = buildSchema(`
  type User { id: ID!, name: String! }
  type Post { id: ID!, authorId: ID!, title: String! }
  type UserAndPosts { user: User, posts: [Post] }
  type Query { userAndPosts(id: ID!): UserAndPosts }
`);
const gatewayRoot = {
  userAndPosts: async ({ id }: any) => {
    const u = (await graphql(schemaA, `{ userById(id: "${id}") { id name } }`, rootA)).data?.userById;
    const p = (await graphql(schemaB, `{ postsByAuthor(authorId: "${id}") { id authorId title } }`, rootB)).data?.postsByAuthor;
    return { user: u, posts: p };
  }
};

const app = express();
app.use(express.json());
app.post('/graphql', async (req, res) => {
  const q = req.body.query;
  const r = await graphql(gatewaySchema, q, gatewayRoot);
  res.json(r);
});

app.listen(8081, () => console.log('Federation gateway listen :8081'));
