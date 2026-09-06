---
date: '2026-07-30T17:30:00+09:00'
draft: true
title: 'Mac に Homebrew で Apache をインストールして起動するまで'
categories: ["Tech"]
tags: ["Homebrew", "Apache", "httpd", "macOS", "launchd"]
summary: "Homebrewでhttpdをインストールし、sudo brew services start httpd でポート80に立ち上げるまでの手順と、Bootstrap failed: 5: Input/output error で失敗し続けた原因の調査記録。"
---

## Homebrew で Apache をインストールする

macOS にはデフォルトで Apache が同梱されているが、OS の初期状態を汚したくない・最新版を使いたいため Homebrew でインストールする。

```bash
brew install httpd
```

## ポート 80 で起動する

`/usr/local/etc/httpd/httpd.conf` の `Listen` をデフォルトの 8080 から 80 に変更する。ポート 80 は 1024 番未満の特権ポートなので、macOS では root 権限がなければ bind できない。そのため起動は `sudo` が必要になる。

```bash
sudo brew services start httpd
```

---

## トラブル: `Bootstrap failed: 5: Input/output error` で起動に失敗し続けた

上記のコマンドを実行したところ、以下の警告とエラーが出て起動に失敗した。

```
Warning: Taking root:admin ownership of some httpd paths:
  /usr/local/Cellar/httpd/2.4.68/bin
  /usr/local/Cellar/httpd/2.4.68/bin/httpd
  /usr/local/opt/httpd
  /usr/local/opt/httpd/bin
  /usr/local/var/homebrew/linked/httpd
This will require manual removal of these paths using `sudo rm` on
brew upgrade/reinstall/uninstall.
Warning: `httpd` must be run as non-root to start at user login!
Bootstrap failed: 5: Input/output error
Error: Failure while executing; `/bin/launchctl bootstrap system /Library/LaunchDaemons/homebrew.mxcl.httpd.plist` exited with 5.
```

何度リトライしても同じエラーになり、`launchctl` のエラーメッセージだけでは原因が全くわからなかったので、ログを一つずつ確認しながら原因を切り分けた。

### 3 つの警告・エラーの意味

1. `Taking root:admin ownership...` — `/usr/local` の Homebrew は通常ユーザー権限でインストールされるため、sudo でサービス化する際に root デーモンとして安全に実行できるよう所有者を変更している。想定内の警告で致命的ではない。
2. `httpd must be run as non-root...` — この plist は本来ユーザー権限で動かす設計だが、sudo を付けたので root(システム)デーモンとして登録しようとしている、という警告。
3. `Bootstrap failed: 5: Input/output error` — これが実際の失敗。ただし launchctl のこのエラーは汎用的すぎて、これ単体では原因がわからない。

### 原因調査

**プロセス一覧を確認**

`ps aux | grep httpd` で確認すると、`kurino`（自分のユーザー）権限で動いている httpd プロセスが 16:40 ごろから残り続けていることがわかった。

```
kurino  49198  ... /usr/local/opt/httpd/bin/httpd
kurino  49200  ... /usr/local/opt/httpd/bin/httpd
(...子プロセス)
```

`lsof` で確認すると、このプロセスは **ポート 8080** で待受けていた。ポート 80 とは競合していない。

**pidfile の中身を確認**

`httpd.conf` の `PidFile` は `/usr/local/var/run/httpd/httpd.pid`。中身を見ると **49198** — まさに上記の古いプロセスの PID が書かれていた。

**httpd バイナリに埋め込まれたメッセージを確認**

`strings` でバイナリを見ると次の文字列が見つかった。

```
httpd (pid %d) already running
```

これで確信が持てた。Apache(httpd) は起動時、`ErrorLog` を開くよりも前に次の処理を行う。

1. `PidFile` を読む
2. そこに書かれた PID に対して `kill(pid, 0)` で生死確認
3. **生きていれば** `httpd (pid %d) already running` と **stderr に直接** print して即終了する

このメッセージは `ap_log_error` 経由ではなく生の `fprintf` で stderr に出力される。plist に `StandardErrorPath` の指定がなかったため、このメッセージはどこにも記録されず、`error_log` にも一切残らなかった。実際、unified log でも新しい httpd プロセスは spawn からわずか 0.6 秒後に終了しており、これと一致する。

**結論: `Bootstrap failed` の原因はポート 80 の競合ではなく、pidfile の生存チェックに引っかかっていたことだった。**

### 解決方法

古いプロセスを止める。自分のユーザー権限で動いていたプロセスなので sudo は不要。

```bash
kill 49198
```

Apache は `SIGTERM` を受けると正常シャットダウン処理を行い、`error_log` に以下を記録した上で自分で pidfile を削除する。

```
[mpm_prefork:notice] caught SIGTERM, shutting down
```

pidfile が消えたことで衝突要因がなくなり、改めて実行した `sudo brew services start httpd` は成功した。

```bash
curl -I http://localhost/
# HTTP/1.1 200 OK
```

### 補足: なぜ sudo なしのプロセスは動けていたのに `http://localhost/` は開けなかったのか

sudo なしのプロセスは **ポート 8080** で待受けていた。`http://localhost/` はポートを省略するとデフォルトで**ポート 80** にアクセスする。ポート 80 は特権ポートなので sudo なしでは絶対に bind できない(bind できなければ即エラー終了になり、8080 に切り替わって動くようなことは起きない)。つまりこの 8080 番の古いプロセスは、`Listen 80` に設定変更する前の別タイミングで起動されたまま残っていたものだった。

- sudo なし → ポート 80 は bind 不可 → 8080 の古いプロセスが残っていた
- `http://localhost/`(=80番)にアクセスしても誰もいない → 接続拒否
- sudo 付きプロセスが 80 番を掴んで初めて `http://localhost/` が繋がった

### 再発防止策

根本原因は「sudo なしで起動した httpd」と「sudo ありで起動した httpd」を同時に生かしてしまい、同じ `ServerRoot`/`PidFile` を共有していたこと。

- httpd の起動方法をどちらかに統一する(ポート 80 を使うなら sudo 運用に統一)
- `sudo brew services start httpd` の前に必ず生存チェックをする

  ```bash
  ps aux | grep '[h]ttpd'
  ```

- 手軽な安全策として、起動前に毎回次を流すのもよい

  ```bash
  pkill -u "$(whoami)" -x httpd 2>/dev/null
  sudo brew services restart httpd
  ```

- `brew services list` は Homebrew が管理していない(手動や launchd 管理外で残った)プロセスを表示しないため、これが "none" と出ていても実プロセスが残っていることがある
