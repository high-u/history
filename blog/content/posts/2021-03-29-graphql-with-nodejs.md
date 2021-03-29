---
title: "Node.js (express 使用) で GraphQL Server を書いてみた"
date: 2021-03-28T12:34:00+09:00
draft: false
---

## 構築する

```bash
node -v
# v14.16.0

echo "{}" > package.json
npm i express express-graphql @graphql-tools/schema
```

./server.js

```js
"use strict"

const express = require("express")
const { graphqlHTTP } = require("express-graphql")
const { makeExecutableSchema } = require("@graphql-tools/schema")

const posts = [
  {
    id: "p1",
    title: "Getting started with Node.js",
    author: "Smith",
    tags: ["t1"]
  },
  {
    id: "p2",
    title: "Getting started with GraphQL",
    author: "Smith",
    tags: ["t1", "t2"]
  },
  {
    id: "p3",
    title: "Getting started with GraphQL Client",
    author: "Joe",
    tags: ["t1", "t2", "t3"]
  }
]

const tags = [
  {
    id: "t1",
    name: "nodejs"
  },
  {
    id: "t2",
    name: "graphql"
  },
  {
    id: "t3",
    name: "react"
  }
]

const schema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type Post {
      id: ID
      title: String
      author: String
      tags: [Tag]
    }
    type Tag {
      id: ID
      name: String
    }
    type Query {
      posts(author: String): [Post]
    }
  `,
  resolvers: {
    Query: {
      posts: (_, args) => {
        console.log("args", args)
        if (args.author) {
          return posts.filter(post => args.author === post.author)
        } else {
          return posts
        }
      }
    },
    Post: {
      tags: (post) => {
        console.log("post", post)
        return tags.filter(tag => post.tags.includes(tag.id))
      }
    }
  }
})

const app = express()
app.use(
  graphqlHTTP({
    schema,
    graphiql: true
  })
)

app.listen(4000, () => {
  console.info("Listening on http://localhost:4000")
})
```

## 起動

```js
node server.js
```

```
open http://localhost:4000/graphql
```

## クエリを投げる

Query

```graphql
query getPosts($postAuthor: String){
  posts(author: $postAuthor){
    id
    title
    author
    tags {
      id
      name
    }
  } 
}
```

Variables

```json
{
  "postAuthor": "Smith"
}
```
