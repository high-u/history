---
title: "Node.js (express 使用) で GraphQL Server を書いてみた"
date: 2021-03-30T20:16:00+09:00
draft: false
---

## 構築する

```bash
node -v
# v14.16.0

echo "{}" > package.json
npm i express express-graphql @graphql-tools/schema better-sqlite3
```

./server.js

```js
"use strict"

const express = require("express")
const { graphqlHTTP } = require("express-graphql")
const { makeExecutableSchema } = require("@graphql-tools/schema")
const DataLoader = require("dataloader")

const db = require("better-sqlite3")("foobar.db");
(() => {
  db.exec("drop table if exists post")
  db.exec(`
    create table if not exists post (
      id text primary key,
      title text,
      author text
    )
  `)
  db.exec("drop table if exists tag")
  db.exec(`
    create table if not exists tag (
      id text primary key,
      name text
    )
  `)
  db.exec("drop table if exists post_tag")
  db.exec(`
    create table if not exists post_tag (
      id text primary key,
      post_id text,
      tag_id text
    )
  `)

  const insertPosts = db.prepare(`
    insert into post (id, title, author)
    values (@id, @title, @author)
  `)
  const insertPostsTran = db.transaction((posts) => {
    for (const post of posts) insertPosts.run(post)
  });
  insertPostsTran([
    {id: "p1", title: "Getting started with Node.js", author: "Smith" },
    {id: "p2", title: "Getting started with GraphQL", author: "Smith" },
    {id: "p3", title: "Getting started with GraphQL Client", author: "Joe" },
  ])

  const insertTags = db.prepare(`
    insert into tag (id, name) values (@id, @name)
  `)
  const insertTagsTran = db.transaction((tags) => {
    for (const tag of tags) insertTags.run(tag)
  });
  insertTagsTran([
    {id: "t1", name: "nodejs"},
    {id: "t2", name: "graphql"},
    {id: "t3", name: "react"},
    {id: "t4", name: "vue"}
  ])

  const insertPostsTags = db.prepare(`
    insert into post_tag (id, post_id, tag_id)
    values (@id, @post_id, @tag_id)
  `)
  const insertPostsTagsTran = db.transaction((postsTags) => {
    for (const postTag of postsTags) insertPostsTags.run(postTag)
  });
  insertPostsTagsTran([
    {id: "pt1", post_id: "p1", tag_id: "t1"},
    {id: "pt2", post_id: "p2", tag_id: "t1"},
    {id: "pt3", post_id: "p2", tag_id: "t2"},
    {id: "pt4", post_id: "p3", tag_id: "t1"},
    {id: "pt5", post_id: "p3", tag_id: "t2"},
    {id: "pt6", post_id: "p3", tag_id: "t3"},
    {id: "pt7", post_id: "p4", tag_id: "t4"}
  ])

  const postRows = db.prepare("SELECT * FROM post").all()
  console.log(postRows)

  const tagRows = db.prepare("SELECT * FROM tag").all()
  console.log(tagRows)

  const postTagRows = db.prepare("SELECT * FROM post_tag").all()
  console.log(postTagRows)
})()

const getAllPosts = () => {
  return db.prepare("SELECT * FROM post").all()
}
const getPostsByAuthor = (author) => {
  return db.prepare("SELECT * FROM post WHERE author = ?").all(author)
}
const getTagsByPostIds = (postIds) => {
  console.log("postIds", postIds)
  const placeholders = new Array(postIds.length).fill("?").join(",")
  const rows = db.prepare(`
    select * from post_tag
    left outer join tag on tag.id = post_tag.tag_id
    where post_id in ( ${placeholders} )`).all(postIds)
  console.log(rows)
  return rows
}

const batchGetTagByPostId = async (ids) => {
  console.log("batchGetTagByPostId", ids)

  // ids (postIds) に紐づく tag を一括で取得する
  const tags = getTagsByPostIds(ids)

  // ids (postIds) 毎に紐づく tag をマッピングする
  return ids.map(id => {
    return tags.filter(tag => id === tag.post_id)
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
          return getPostsByAuthor(args.author)
        } else {
          return getAllPosts()
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

## 所感

前回となんら変わりは無いのだけどね。少し実際のサーバに近づけてみた。そして、今後やりたいことへの第一歩。  
やりたいことは、 [hasura](https://hasura.io/) のようなものを作る場合の検証というか、どんな考慮が必要なのかを、体感したい。  
ここ [Modelling many-to-many table relationships | Hasura GraphQL Docs](https://hasura.io/docs/latest/graphql/core/guides/data-modelling/many-to-many.html) に、課題になりそうなことの hasura の考えが載っていた。すごく参考になる。  
現状の考えでいけば、まずは view での対応を考えてみる。 hasura の推奨では無い方だけど。
