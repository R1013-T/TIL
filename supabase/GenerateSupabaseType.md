# Generate Supabase Types

Supabase で定義したテーブルから TypeScript の型定義を生成する。

1. ログイン

```shell
npx supabase login
```

2. テーブル定義を取得

```shell
npx supabase init
```

3. リファレンス ID を設定

```shell
npx supabase link --project-ref your_project_id
```

4. 型定義を生成

```shell
npx supabase gen types typescript --linked > database.types.ts
```
