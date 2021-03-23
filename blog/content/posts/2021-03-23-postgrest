---
title: "PostgREST 弄ってみた"
date: 2021-03-24T00:49:00+09:00
draft: false
---

# PostgREST 弄ってみた

## 起動と停止

```bash
docker-compose up -d
docker-compose down --volumes --remove-orphans
```

## ファイル

`./.env`

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=postgres

DB_ANON_ROLE=anon
#DB_ANON_ROLE=postgres
DB_SCHEMA=public
```

`docker-compose.yaml`

```yaml
version: "3"
services:
  postgres:
    container_name: postgres
    user: "root:root"
    image: postgres:11-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
      - DB_ANON_ROLE=${DB_ANON_ROLE}
      - DB_SCHEMA=${DB_SCHEMA}
    volumes:
      - ./initdb:/docker-entrypoint-initdb.d
    networks:
      - postgrest-backend
    restart: always

  postgrest:
    container_name: postgrest
    image: postgrest/postgrest:latest
    ports:
      - "3000:3000"
    environment:
      - PGRST_DB_URI=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      - PGRST_DB_SCHEMA=${DB_SCHEMA}
      - PGRST_DB_ANON_ROLE=${DB_ANON_ROLE}
    networks:
      - postgrest-backend
    restart: always

networks:
  postgrest-backend:
    driver: bridge
```

`./initdb/create-anon-role.sh`

見れば分かる通りスーパーユーザーの権限を付与しているので、わざわざユーザーを作らずに、既に存在する postgres ユーザーを利用しても良いのだが、postgres 起動時にこのシェルを実行させるのにハマったので取っておくことにした。
認証したくなった時に登場することになるし。
ハマった内容の解決は、docker-compose.yaml の `user: "root:root"` の部分。

```bash
#!/bin/bash

psql -U ${POSTGRES_USER} -d ${POSTGRES_DB} <<-EOSQL
  CREATE USER ${DB_ANON_ROLE};
  -- GRANT USAGE ON SCHEMA ${DB_SCHEMA} TO ${DB_ANON_ROLE};
  -- ALTER DEFAULT PRIVILEGES   IN SCHEMA ${DB_SCHEMA} GRANT ALL ON TABLES TO ${DB_ANON_ROLE};
  -- GRANT ALL ON ALL TABLES    IN SCHEMA ${DB_SCHEMA} TO ${DB_ANON_ROLE};
  -- GRANT ALL ON ALL SEQUENCES IN SCHEMA ${DB_SCHEMA} TO ${DB_ANON_ROLE};
  -- GRANT ALL ON ALL FUNCTIONS IN SCHEMA ${DB_SCHEMA} TO ${DB_ANON_ROLE};
  ALTER ROLE ${DB_ANON_ROLE} WITH SUPERUSER;
EOSQL
```

```bash
chmod +x ./initdb/create-anon-role.sh
```

`./initdb/todos.sql`

```sql
create table todos (
  id serial primary key,
  done boolean not null default false,
  task text not null,
  due timestamptz
);

insert into todos (task) values
  ('finish tutorial 0'), ('pat self on back');
```

## API を叩く

```bash
curl -s http://localhost:3000/todos | jq .

curl http://localhost:3000/todos -X POST \
  -H "Content-Type: application/json"
  -d '{"task": "do bad thing"}'

curl -s http://localhost:3000/todos | jq .
```
