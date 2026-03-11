---
date: '2026-02-05T09:00:00+09:00'
draft: false
title: 'TypeScript ジェネリクス入門 - <T>って何？を初心者向けに解説'
categories: ["Tech"]
tags: ["TypeScript", "ジェネリクス", "型"]
summary: "TypeScriptの<T>（ジェネリクス）が何なのか、初心者向けにわかりやすく解説します。"
---

## はじめに

TypeScriptのコードを読んでいると、こんな記法を見かけませんか？

```typescript
function example<T>(value: T): T {
  return value;
}
```

この `<T>` が何なのかわからず、モヤモヤしている人も多いのではないでしょうか。

この記事では、**ジェネリクス（Generics）** の基本を初心者向けに解説します。

## ジェネリクスとは？

ジェネリクスを一言で説明すると、**「型の変数」** です。

普通の変数が「値」を入れる箱だとすると、ジェネリクスは「型」を入れる箱です。

```typescript
// 普通の変数：値を入れる
let value = 100;

// ジェネリクス：型を入れる
// <T> の T に string や number などの型が入る
```

## なぜジェネリクスが必要なの？

### 問題：型ごとに関数を作るのは大変

「渡された値をそのまま返す」関数を作りたいとします。

```typescript
// number用
function returnNumber(value: number): number {
  return value;
}

// string用
function returnString(value: string): string {
  return value;
}

// boolean用
function returnBoolean(value: boolean): boolean {
  return value;
}
```

型ごとに同じような関数を作るのは面倒ですよね。

### 解決策1：any型を使う（ダメな例）

```typescript
function returnAny(value: any): any {
  return value;
}

const result = returnAny("hello");
// result の型は any... 型安全じゃない！
```

`any` を使うと、戻り値の型情報が失われてしまいます。

### 解決策2：ジェネリクスを使う（良い例）

```typescript
function returnValue<T>(value: T): T {
  return value;
}

const num = returnValue<number>(100);    // num は number型
const str = returnValue<string>("hello"); // str は string型
const bool = returnValue(true);           // bool は boolean型（型推論）
```

ジェネリクスを使えば、**型安全を保ちながら**、様々な型に対応できます。

## 基本的な書き方

### 関数のジェネリクス

```typescript
// 基本形
function 関数名<T>(引数: T): T {
  return 引数;
}

// 使い方
関数名<string>("hello");  // 明示的に型を指定
関数名("hello");          // 型推論に任せる（省略可）
```

`T` は慣習的に使われる名前ですが、何でもOKです。

```typescript
function example<Type>(value: Type): Type { ... }
function example<U>(value: U): U { ... }
function example<TData>(value: TData): TData { ... }
```

### よく使われる名前

| 名前 | 意味 | 使用例 |
|------|------|--------|
| `T` | Type（型） | 一般的な型 |
| `K` | Key（キー） | オブジェクトのキー |
| `V` | Value（値） | オブジェクトの値 |
| `E` | Element（要素） | 配列の要素 |
| `R` | Return（戻り値） | 関数の戻り値 |

## 実践例

### 例1：配列の最初の要素を取得

```typescript
function getFirst<T>(arr: T[]): T | undefined {
  return arr[0];
}

const nums = [1, 2, 3];
const first = getFirst(nums);  // first は number | undefined

const strs = ["a", "b", "c"];
const firstStr = getFirst(strs);  // firstStr は string | undefined
```

### 例2：2つの値をペアにする

```typescript
function makePair<T, U>(first: T, second: U): [T, U] {
  return [first, second];
}

const pair = makePair("name", 100);
// pair の型は [string, number]
```

複数のジェネリクスを使う場合は、カンマで区切ります。

### 例3：型に制約をつける（extends）

特定の型だけに限定したい場合は `extends` を使います。

```typescript
// T は { length: number } を持つ型に限定
function getLength<T extends { length: number }>(value: T): number {
  return value.length;
}

getLength("hello");     // OK: stringはlengthを持つ
getLength([1, 2, 3]);   // OK: 配列はlengthを持つ
getLength(100);         // エラー: numberはlengthを持たない
```

## 型（インターフェース）のジェネリクス

関数だけでなく、型定義にもジェネリクスが使えます。

### 例1：APIレスポンスの型

```typescript
// 汎用的なAPIレスポンス型
type ApiResponse<T> = {
  data: T;
  status: number;
  message: string;
};

// User用のレスポンス
type User = { id: number; name: string };
type UserResponse = ApiResponse<User>;
// {
//   data: { id: number; name: string };
//   status: number;
//   message: string;
// }

// Product用のレスポンス
type Product = { id: number; price: number };
type ProductResponse = ApiResponse<Product>;
```

### 例2：Reactのステート（useState）

Reactを使ったことがある人は、これを見たことがあるはずです。

```typescript
const [count, setCount] = useState<number>(0);
const [name, setName] = useState<string>("");
const [user, setUser] = useState<User | null>(null);
```

`useState` はジェネリクスを使って、ステートの型を指定しています。

## TypeScript標準のジェネリクス型

TypeScriptには便利なジェネリクス型が用意されています。

```typescript
// Array<T>: 配列
const nums: Array<number> = [1, 2, 3];
// number[] と同じ

// Promise<T>: 非同期処理の結果
const promise: Promise<string> = fetch("/api").then(r => r.text());

// Partial<T>: すべてのプロパティをオプショナルに
type PartialUser = Partial<User>;

// Record<K, V>: キーと値の型を指定したオブジェクト
const scores: Record<string, number> = {
  math: 90,
  english: 85
};
```

## ジェネリクスの読み方

コードを読むときのコツです。

```typescript
function map<T, U>(arr: T[], fn: (item: T) => U): U[]
```

1. `<T, U>` → 「TとUという2つの型変数を使う」
2. `arr: T[]` → 「arrはT型の配列」
3. `fn: (item: T) => U` → 「fnはTを受け取ってUを返す関数」
4. `: U[]` → 「戻り値はU型の配列」

つまり「T型の配列を受け取り、各要素をU型に変換した配列を返す」関数です。

## まとめ

ジェネリクスのポイントをまとめます。

- ジェネリクス（`<T>`）は**型の変数**
- **型安全を保ちながら**、様々な型に対応できる
- `any` を使わずに柔軟な関数・型が作れる
- `T` は慣習的な名前。`Type` など好きな名前でOK
- `extends` で型に制約をつけられる

最初は難しく感じるかもしれませんが、「型を入れる箱」と考えれば理解しやすくなります。

## 理解度チェック

### 問1

ジェネリクスを一言で説明すると何ですか？

<details>
<summary>答え</summary>
型の変数。値ではなく「型」を入れる箱のようなもの。
</details>

### 問2

以下のコードで `result` の型は何になりますか？

```typescript
function identity<T>(value: T): T {
  return value;
}
const result = identity("hello");
```

<details>
<summary>答え</summary>
string型。"hello"という文字列を渡しているので、Tはstringと推論される。
</details>

### 問3

`any` ではなくジェネリクスを使うメリットは何ですか？

<details>
<summary>答え</summary>
型情報が保持され、型安全が保たれる。anyを使うと戻り値などの型情報が失われてしまう。
</details>

### 問4

`<T extends string>` はどういう意味ですか？

<details>
<summary>答え</summary>
型Tはstring型（またはstringのサブタイプ）に限定される、という制約。
</details>
