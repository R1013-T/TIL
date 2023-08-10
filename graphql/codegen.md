# codegen

## GraphQL Code Generator

GraphQL Code Generator は GraphQL のスキーマから TypeScript の型定義や、GraphQL のクエリを実行するためのコードを生成するツールです。

### Install

```bash
npm i -D @graphql-codegen/cli
npm i -D @graphql-codegen/typescript
```

### init

```bash
npx graphql-codegen init
```

- 使用ライブラリや、URL、出力先などを選択する。

### Add queries in queries/queries.ts file

[参考ファイル](https://github.com/GomaGoma676/nextjs-hasura-basic-lesson/blob/main/queries/queries.ts)

queries/queries.ts

```ts
import {gql} from "@apollo/client";

export const GET_USERS = gql`
  query GetUsers {
    users(order_by: { created_at: desc }) {
      id
      name
      created_at
    }
  }
`;

// ...
```

### Generate

- graphql.tsx が生成される。

```bash
npx graphql-codegen
```
