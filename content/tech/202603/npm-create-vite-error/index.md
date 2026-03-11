---
date: '2026-03-11T21:39:12+09:00'
draft: true
title: 'npm create vite@latest 実行で出たエラーを解決した'
categories: ["Tech"]
tags: ["npm", "Vite", "TypeScript", "VSCode", "cli"]
summary: "npm create vite@latest でReactプロジェクトを作成したところVSCode上でTypeScriptのエラーが発生。原因はVSCode内蔵のTypeScriptバージョンの古さにありました。"
---

## はじめに

手っ取り早くReactアプリを作りたかったので `npm create vite@latest` を実行したのですが、VSCode上でエラーが出ました。
ネットで同じエラーの人が見つからず自分の環境固有の問題かと思いましたが、原因を調べると仕組みの話で納得感のある解決ができたので記録しておきます。

## エラー内容

```bash
npm create vite@latest
```

を実行してプロジェクトを作成し、VSCodeで開くと `tsconfig.app.json` と `App.tsx` でエラーが発生していました。

![tsconfig.app.jsonエラー](tsconfigエラー.png)

![App.tsxエラー](apptsx-error.png)

## 原因

### VSCodeは独自のTypeScriptを持っている

そもそも **VSCode自体がTypeScriptで開発されています**。VSCodeのソースコードはTypeScriptで書かれており、TypeScriptはVSCodeにとって開発言語そのものです。

さらにVSCodeは、エディタ上で型チェックや補完を動かすために **TypeScript Language Service**（TypeScriptの型解析エンジン）を内部に同梱しています。これがエディタ機能の核であり、プロジェクトの `node_modules` とは完全に別物です。

```
VSCode本体
  ├── VSCode自身のコード（TypeScriptで書かれている）
  └── TypeScript Language Service（同梱）← エディタの型チェックに使われる
        バージョン: 4.9.3

プロジェクト
  └── node_modules/typescript（npm installで入る）
        バージョン: 5.x
```

`npm install` を実行すると `package.json` に記載のTypeScript（5.x系）は `node_modules/typescript` にインストールされます。
しかし **VSCodeのエディタ機能はデフォルトで同梱のTypeScript Language Serviceを使うため、`node_modules` のTypeScriptは参照しません**。

これはVSCodeの意図的な設計で、「プロジェクトにTypeScriptがインストールされていなくてもエディタが動く」ようにするためです。

### なぜエラーになるのか

Viteが生成する `tsconfig.app.json` はTypeScript 5.0以降で追加された設定を使っています。

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler"  // TypeScript 5.0で追加されたオプション
  }
}
```

VSCode同梱の古いTypeScript（4.9.3）はこのオプションを知らないため、エラーとして扱ってしまいます。

![typescriptのバージョンが古かった](vscode-typescript.png)

## 解決方法

VSCodeが参照するTypeScriptを、同梱のものからワークスペース（`node_modules`）のものに切り替えます。

1. コマンドパレットを開く（`Cmd + Shift + P`）
2. 「TypeScript: Select TypeScript Version」を選択
3. 「ワークスペースのバージョンを使用 5.x.x」を選択

これで解決！

