---
date: '2026-02-05T10:00:00+09:00'
draft: false
title: 'TypeScript Partial型入門 - オブジェクトのプロパティをすべてオプショナルにする'
categories: ["Tech"]
tags: ["TypeScript", "型", "Utility Types"]
summary: "TypeScriptのPartial型について初心者向けに解説します。使い方、実践例、仕組みまでわかりやすく説明します。"
---

## はじめに

TypeScriptには、型を便利に変換できる**Utility Types（ユーティリティ型）** が用意されています。その中でも特によく使われるのが `Partial<T>` です。

この記事では、Partial型の基本から実践的な使い方まで、初心者向けに解説します。

## Partial型とは？

`Partial<T>` は、オブジェクト型のすべてのプロパティを**オプショナル（省略可能）** にする型です。

```typescript
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

簡単に言うと、「**全部のプロパティに `?` をつける**」ということです。

## 基本的な使い方

### Before: Partial なし

```typescript
type User = {
  id: number;
  name: string;
  email: string;
};

// すべてのプロパティが必須
const user: User = {
  id: 1,
  name: "田中太郎",
  email: "tanaka@example.com"
};
```

### After: Partial あり

```typescript
type User = {
  id: number;
  name: string;
  email: string;
};

// すべてのプロパティがオプショナルになる
const partialUser: Partial<User> = {
  name: "田中太郎"
  // id と email は省略OK
};
```

`Partial<User>` は以下と同じ意味になります。

```typescript
type PartialUser = {
  id?: number;
  name?: string;
  email?: string;
};
```

## 実践的なユースケース

### 1. 更新処理（PATCH）

APIでデータを部分更新するとき、Partial型が活躍します。

```typescript
type User = {
  id: number;
  name: string;
  email: string;
  age: number;
};

// 更新したいフィールドだけ渡せる
function updateUser(id: number, updates: Partial<User>) {
  // データベースを更新する処理
  console.log(`Updating user ${id}:`, updates);
}

// 名前だけ更新
updateUser(1, { name: "新しい名前" });

// メールと年齢を更新
updateUser(2, { email: "new@example.com", age: 30 });
```

### 2. デフォルト値とのマージ

設定オブジェクトでよく使われるパターンです。

```typescript
type Config = {
  theme: "light" | "dark";
  fontSize: number;
  showSidebar: boolean;
};

const defaultConfig: Config = {
  theme: "light",
  fontSize: 14,
  showSidebar: true
};

function createConfig(options: Partial<Config>): Config {
  return { ...defaultConfig, ...options };
}

// 一部だけ指定すればOK
const myConfig = createConfig({ theme: "dark" });
// 結果: { theme: "dark", fontSize: 14, showSidebar: true }
```

### 3. フォームの状態管理

フォームの入力状態を管理するときにも便利です。

```typescript
type FormData = {
  username: string;
  email: string;
  password: string;
};

// 初期状態は空でもOK
const formState: Partial<FormData> = {};

// ユーザーが入力するたびに更新
formState.username = "user123";
formState.email = "user@example.com";
```

### 4. テストデータの作成

テストで一部のプロパティだけ指定したいときに役立ちます。

```typescript
type Product = {
  id: number;
  name: string;
  price: number;
  description: string;
  category: string;
};

function createTestProduct(overrides: Partial<Product> = {}): Product {
  return {
    id: 1,
    name: "テスト商品",
    price: 1000,
    description: "テスト用の商品です",
    category: "テスト",
    ...overrides
  };
}

// 価格だけ変えたテストデータを作成
const expensiveProduct = createTestProduct({ price: 99999 });
```

## Partial型の注意点

### 1. ネストしたオブジェクトには効かない

Partial型は**1階層目のプロパティのみ**オプショナルにします。

```typescript
type User = {
  id: number;
  profile: {
    name: string;
    age: number;
  };
};

type PartialUser = Partial<User>;
// 結果:
// {
//   id?: number;
//   profile?: {
//     name: string;  ← オプショナルにならない
//     age: number;   ← オプショナルにならない
//   };
// }
```

深い階層もオプショナルにしたい場合は、**DeepPartial**を自作する必要があります。

```typescript
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};
```

### 2. undefinedが許容される

オプショナルプロパティは `undefined` を許容します。

```typescript
const user: Partial<User> = {
  name: undefined  // これもOK
};
```

## 関連するUtility Types

Partial型と一緒に覚えておくと便利な型です。

| 型 | 説明 |
|----|------|
| `Partial<T>` | すべてのプロパティをオプショナルに |
| `Required<T>` | すべてのプロパティを必須に（Partialの逆） |
| `Pick<T, K>` | 指定したプロパティだけを抽出 |
| `Omit<T, K>` | 指定したプロパティを除外 |

### 組み合わせの例

```typescript
type User = {
  id: number;
  name: string;
  email: string;
};

// idは必須、他はオプショナル
type UserUpdate = Pick<User, "id"> & Partial<Omit<User, "id">>;

const update: UserUpdate = {
  id: 1,           // 必須
  name: "新しい名前" // オプショナル
};
```

## まとめ

Partial型のポイントをまとめます。

- `Partial<T>` はすべてのプロパティを**オプショナル**にする
- **更新処理**や**デフォルト値のマージ**で特に便利
- **ネストしたオブジェクト**には効かない（1階層のみ）
- `Required<T>` は逆の動作（すべて必須に）

Partial型を使いこなせると、より柔軟で型安全なコードが書けるようになります。

## 理解度チェック

### 問1

`Partial<T>` は何をする型ですか？

<details>
<summary>答え</summary>
オブジェクト型Tのすべてのプロパティをオプショナル（省略可能）にする。
</details>

### 問2

以下のコードで `partialUser` に代入できる値はどれですか？

```typescript
type User = { id: number; name: string; };
const partialUser: Partial<User> = ???;
```

A. `{}`
B. `{ id: 1 }`
C. `{ name: "太郎" }`
D. すべて正解

<details>
<summary>答え</summary>
D. すべて正解。Partial型ではすべてのプロパティがオプショナルになるため、空オブジェクトも含めすべて有効。
</details>

### 問3

Partial型がネストしたオブジェクトに対して「浅い」と言われる理由は何ですか？

<details>
<summary>答え</summary>
1階層目のプロパティのみオプショナルになり、ネストしたオブジェクト内部のプロパティはオプショナルにならないため。
</details>
