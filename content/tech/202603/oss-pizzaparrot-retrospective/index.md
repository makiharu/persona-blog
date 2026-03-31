---
date: '2026-03-19T00:00:00+09:00'
draft: false
title: '初OSSコントリビュート振り返り〜WholePizza ParrotのPRで学んだこと'
categories: ["Tech"]
tags: ["OSS", "GIF", "Python", "PIL", "gifsicle", "Party Parrot"]
summary: "cult of the party parrot に自作GIFを投稿し、レビュアーとやり取りしながらPRをマージするまでの体験記。GIF透過処理やgifsicle最適化のハマりどころと、OSSコミュニケーションで学んだことをまとめます。"
---

## はじめに

[cult of the party parrot](https://github.com/jmhobbs/cultofthepartyparrot.com) に自作GIFを投稿し、レビュアーとやり取りしながらPRをマージに持っていくまでの体験記です。

この記事では以下をまとめます。

- 初OSSコントリビュートの流れ
- GIF制作でハマった技術的ポイント
- OSS特有のレビューコミュニケーションで学んだこと

---

## 背景

会社メンバーとの雑談で「Party Parrot」を知り、以下の記事を読んで興味を持ちました。

https://qiita.com/ayatothos/items/0c3da3d48fe8ade705e6

この記事をきっかけに「自分でもOSSに貢献できるかもしれない」と思い、新しいParrot GIFを作ることにしました。

---

## 作成について

作品の方向性やアイデアは自分の発想ですが、GIF制作の具体的な実装はAIと相談しながら進めました。

---

## レビュー対応について

最初は「Pizza Parrot（HD版）」としてPRを提出しましたが、レビュアーから以下のコメントがありました。

> Looks great! I wonder if we should disambiguate names with the existing 🍕 Parrot.
> Generally the HD versions recreate the SD versions without any meaningful changes.
> I like the parrot eating the whole pizza though, maybe we could rename that to wholepizzaparrot.gif or something?

このコメントをもとに修正対応を行いました。

---

## ハマったこと1: GIFの透過処理

### 問題

よくよく見ると、最初に投稿したGIFは背景が透過されていませんでした。痛恨のミスです。
既存の `pirateparrot.gif` などと比較すると明らかに違和感がありました。

### 原因

フレーム生成時に背景を白で塗りつぶしていたことが原因でした。

```python
# NG: 白背景
frame = Image.new("RGBA", (CANVAS_SIZE, CANVAS_SIZE), (255, 255, 255, 255))

# OK: 透明背景
frame = Image.new("RGBA", (CANVAS_SIZE, CANVAS_SIZE), (0, 0, 0, 0))
```

### GIF透過の仕組みと注意点

GIFは「特定のパレットインデックスを透明として扱う」仕様です。

そのため、RGBA → GIF変換時に透過情報が失われないように処理が必要になります。

```python
def rgba_to_gif_frame(frame, global_palette):
    alpha = np.array(frame.getchannel("A"))
    p = frame.convert("RGB").quantize(palette=global_palette, dither=0)
    orig_palette = p.getpalette()

    arr = np.array(p, dtype=np.uint8)
    arr = (arr + 1).clip(0, 255).astype(np.uint8)

    arr[alpha < 128] = 0

    new_palette = [0, 0, 0] + orig_palette[:255 * 3]
    result = p.copy()
    result.putdata(arr.flatten().tolist())
    result.putpalette(new_palette)
    return result
```

さらに保存時に以下を指定する必要があります。

```python
transparency=0, disposal=2
```

---

## ハマったこと2: gifsicle最適化で頭が消える

### 問題

`gifsicle -O3` で最適化した結果、アニメーション中にパロットの頭が消える問題が発生しました。

### 原因

`-O3` は差分フレームのみを保存する最適化を行います。

しかし `disposal=background` と組み合わさると、前フレームで消された部分が再描画されないケースが発生します。

```
frame 0: 128x128（全体）
frame 5: 118x71 at (10, 54)
→ 頭（row 43）が範囲外で消える
```

### 解決策

全フレームをフルサイズに展開しました。

```python
subprocess.run(["gifsicle", "--unoptimize", abs_out, "-o", abs_tmp])
```

---

## ハマったこと3: ピザがキャンバスからはみ出す

### 問題

ピザが128×128のキャンバスからはみ出していました。

```
PIZZA_CENTER = (113, 84), PIZZA_SIZE = 58
→ 右端: 142px > 128px
```

### 解決策

キャンバスを拡張し、位置調整しました。

```python
CANVAS_SIZE   = 160
PARROT_OFFSET = (0, 16)
PIZZA_CENTER  = (113, 100)
```

---

## OSSコントリビューションで学んだこと

### 1. レビュアーの意図を正確に読む

- HD版は既存の再現 → 名前衝突NG
- 新しい動き → 新パロットとして扱うべき

最初はHDとして配置しましたが、コンセプト的に誤りでした。

### 2. PRは最終状態が重要

試行錯誤の履歴はローカルに残しつつ、PRは整理して見やすくすることが重要です。

今回のリポジトリでは「1PR = 1コミット」が一般的でした。

### 3. レビュー返信はシンプルに

対応内容を簡潔に伝えることで、レビュアーの確認コストを下げられます。

---

## まとめ

GIFの透過、gifsicleの最適化、キャンバス設計など技術的な調整は多くありました。

ただ、それ以上に印象に残ったのは「OSS活動の楽しさ」です。

海外の開発者と直接やり取りできる体験は純粋に楽しく、OSSは「技術力が高い人だけの世界」ではなく、

- ルールを理解する
- 丁寧にコミュニケーションする

これだけでも十分に価値があると感じました。

次はもう少し難しいコントリビュートにも挑戦してみようと思います。
