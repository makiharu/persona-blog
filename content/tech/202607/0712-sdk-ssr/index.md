---
date: '2026-07-12T10:00:00+09:00'
draft: false
title: 'Next.jsのSSR・SSG・Hydrationを整理して理解した'
categories: ["Tech"]
tags: ["Next.js", "SSR", "SSG", "Hydration", "React"]
summary: "Next.jsプロジェクトへSDKを組み込む際にdynamic importとssr: falseが必要になったことをきっかけに、SSR・SSG・Hydrationの仕組みを整理した。"
---

あるSDKをNext.jsプロジェクトへ組み込む際、`dynamic import` と `ssr: false` が必要になった。

「なぜSSRを無効化する必要があるのか？」を調べる中で、SSRやHydrationの仕組みについて理解が深まったので整理する。

## 一般的なWebアプリの構成

通常のSPA（Reactのみ）の場合は次のような構成になる。

```text
ブラウザ
    │ HTTP
    ▼
APIサーバー
    │
    ▼
　DB
```

ブラウザ側でReactが動き、必要なデータだけをAPIサーバーから取得する。

HTMLはブラウザ側で生成される。

## SSRを使うNext.jsの構成

SSRを利用する場合は、Next.jsサーバーが追加される。

```text
ブラウザ
    │
    ▼
Next.jsサーバー(Node.js)
    │
    ▼
APIサーバー
    │
    ▼
DB
```

このとき、

1. ブラウザはNext.jsサーバーへアクセスする
2. Next.jsサーバーがReactコンポーネントを実行する
3. HTMLを生成する
4. APIサーバーから必要なデータを取得する
5. HTMLをブラウザへ返す

という流れになる。

つまり、

**HTMLを返しているのはAPIサーバーではなく、Next.jsサーバー。**

## Next.jsサーバーとは

> 「Next.jsサーバーって誰がどうやって用意するの？」

例えば本番サーバーへデプロイして

```bash
next start
```

するとNode.js上でNext.jsが起動し、これが「Next.jsサーバー」になる。

別途Node.js用のプロジェクトを作るわけではない。

## SSRサーバーを持たない構成もある

Next.jsは必ずNode.jsサーバーを立てる必要があるわけではなく、`next export`（あるいは`output: 'export'`）で静的エクスポートし、本番ではNext.jsサーバーを動かさずに配信する構成も取れる。

```text
ブラウザ・任意のランタイム
    │
    ▼
事前生成されたHTML + JavaScript
```

この場合、アクセス時にNext.jsサーバーがHTMLを生成しているわけではなく、ビルド時に生成済みの静的ファイルがそのまま配信される。

## なぜSDKでエラーになるのか

ブラウザ向けのSDKの中には

* window
* document

などブラウザのAPIを前提にしているものがある。

しかしビルド中はNode.js環境なので、

```ts
window
```

は存在しない。

そのため

```
ReferenceError: window is not defined
```

のようなエラーになる。

そこで

```ts
dynamic(() => import("./SomeSdkComponent"), {
  ssr: false,
});
```

として、**ブラウザだけでSDKを読み込む**ようにしている。

## Hydrationとは

Hydrationは、サーバー（またはビルド時）で作られたHTMLに、Reactがイベントなどを結び付ける処理のこと。

イメージすると、

```text
① HTML表示
      ↓
② Reactが起動
      ↓
③ ボタンや入力欄が動くようになる
```

という流れになる。

## SSR・SSG・SPAの違い

| 方式  | HTMLを作るタイミング        |
| --- | ------------------- |
| SPA | ブラウザ                |
| SSR | リクエストごとにNext.jsサーバー |
| SSG | ビルド時に事前生成           |

## 今回理解できたこと

* SSRではNext.jsサーバーがHTMLを生成する
* Next.jsサーバーはNext.jsプロジェクト自体が動いているもの
* `next export`（`output: 'export'`）では本番にNext.jsサーバーはいない
* ブラウザ専用のSDKが落ちる理由はNode.js環境には`window`が存在しないため
* `dynamic import + ssr: false`でブラウザだけに読み込みを限定できる
* HydrationはHTMLへReactのイベント処理を結び付ける仕組み


補足すると、`next export` は現在のNext.jsでは非推奨となっており、新しいプロジェクトでは `output: 'export'` を設定して `next build` を実行する方式が推奨されている。ただし、SSR・SSG・Hydrationの考え方自体はこのまとめの内容で問題ない。

## 理解度チェック

理解度チェッククイズ（全4問）

問1

SSRを利用するNext.jsの構成で、実際にHTMLを生成しているのはAPIサーバーとNext.jsサーバーのどちらでしょうか？

<details> <summary>答え</summary> Next.jsサーバー。APIサーバーはデータを返すだけで、そのデータを使ってReactコンポーネントを実行しHTMLへ変換しているのはNext.jsサーバー側の処理。 </details>

問2

ブラウザ向けのSDKをそのままNext.jsに組み込むと、ビルド時や`next start`実行時に`window is not defined`のようなエラーになることがあります。これはなぜでしょうか？

<details> <summary>答え</summary> SSR（サーバーサイドでのReact実行）はNode.js環境で行われ、Node.jsには`window`や`document`といったブラウザAPIが存在しないため。ブラウザAPIを前提にしたSDKのコードがサーバー側でも評価されてしまい、参照エラーになる。 </details>

問3

Hydrationとは何かを簡潔に説明してください。

<details> <summary>答え</summary> サーバー（またはビルド時）で生成済みのHTMLに対して、Reactがイベントハンドラなどを結び付け、ボタンや入力欄が実際に動くようにする処理のこと。 </details>

問4

SPA・SSR・SSGは最終的にはどれもブラウザにHTML/JSを届けますが、何が本質的に違うのでしょうか？「HTMLがいつ・どこで作られるか」という観点で3つを比較し、それぞれの方式がなぜその選択をしているのか（何を優先しているのか）まで考えて説明してください。

<details> <summary>答え</summary> 3つとも「Reactコンポーネントを実行してHTMLに変換する」処理自体は同じだが、それを行うタイミングと場所が異なる。SPAはブラウザが表示するたびに（クライアントで、都度）実行する。SSRはリクエストが来るたびに（サーバーで、都度）実行し、常に最新のデータを反映したHTMLを返せる代わりに毎回サーバーの計算コストがかかる。SSGはビルド時に（サーバー相当の環境で、一度だけ）実行しておき、以降は生成済みのHTMLを配信するだけなので速く安く済むが、データが更新されても再ビルドするまでHTMLには反映されない。つまり3方式の違いは「都度作るか、一度作って使い回すか」「その処理をどこで行うか」というトレードオフの選び方の違いである。 </details>
