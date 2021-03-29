---
title: "GraphQL Server に Dataloader をいれてみた"
date: 2021-03-29T19:54:00+09:00
draft: false
---

## Dataloader 導入前との比較

[Node.js (express 使用) で GraphQL Server を書いてみた | history](https://history.high-u.cf/posts/2021-03-29-graphql-with-nodejs/) をベースに Dataloader を導入した。  
データの持ち方を、tag に配列を使った形から、通常の RDBMS でありそうな感じに変更した。  
四苦八苦したが、 [Building a GraphQL API in JavaScript | Heroku](https://blog.heroku.com/building-graphql-api-javascript#performance-considerations) がすごく参考になった。

## 構築する

```bash
node -v
# v14.16.0

echo "{}" > package.json
npm i express express-graphql @graphql-tools/schema dataloader
```

./server.js

```js
"use strict"

const express = require("express")
const { graphqlHTTP } = require("express-graphql")
const { makeExecutableSchema } = require("@graphql-tools/schema")
const DataLoader = require('dataloader')

const postTable = [
  {
    id: "p1",
    title: "Getting started with Node.js",
    author: "Smith"
  },
  {
    id: "p2",
    title: "Getting started with GraphQL",
    author: "Smith"
  },
  {
    id: "p3",
    title: "Getting started with GraphQL Client",
    author: "Joe"
  }
]

// Dataloader 導入以前は tag マスター的な雰囲気のデータ構造にしていたが
// 導入に際して、post と tag のジャンクションテーブルと tag マスタを join したようなデータ構造かな？
const tagTable = [
  {
    id: "t1",
    postId: "p1",
    name: "nodejs"
  },
  {
    id: "t2",
    postId: "p2",
    name: "nodejs"
  },
  {
    id: "t3",
    postId: "p2",
    name: "graphql"
  },
  {
    id: "t4",
    postId: "p3",
    name: "nodejs"
  },
  {
    id: "t5",
    postId: "p3",
    name: "graphql"
  },
  {
    id: "t6",
    postId: "p3",
    name: "react"
  },
  {
    id: "t7",
    postId: "p4",
    name: "vue"
  }
]

// 仮想 DB の tag テーブルの select クエリのようなもの
// `select * from tag where postid in (?, ?, ?);`
const selectFromTagWherePostIdIn = (postIds) => {
  return tagTable.filter(tag => postIds.includes(tag.postId))
}

const batchGetTagByPostId = async (ids) => {
  console.log("batchGetTagByPostId", ids)

  // 仮想 DB からの取得処理
  // ids (postIds) に紐づく tag を一括で取得する
  const tags = selectFromTagWherePostIdIn(ids)

  // ids (postIds) 毎に紐づく tag をマッピングする
  return ids.map(id => {
    return tags.filter(tag => id === tag.postId)
  })
}
// variables を変更してクエリを何度か実行しながら、ログを見ると
// バッチ処理になっているのと、キャッシュされているのが分かる。
const tagLoader = new DataLoader(batchGetTagByPostId)

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
          return postTable.filter(post => args.author === post.author)
        } else {
          return postTable
        }
      }
    },
    Post: {
      tags: (post) => {
        return tagLoader.load(post.id);
      },
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

## 起動する

```bash
node server.js
```

GraphiQL を開く

```bash
open http://localhost:4000/graphql
```

## クエリを投げる

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
