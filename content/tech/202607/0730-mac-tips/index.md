
## Mac開発環境

これは入れておけツール

Nix
Docker
アルフレッド？
なんかもっと違うのがあったような


Mac開発で一番最初にやること

隠しファイル・フォルダを表示状態にする
Finder上で常に見えるようにしたい場合は、隠しファイルの表示ショートカットを使います。
1. Finderで 「Macintosh HD」（またはユーザーのルートフォルダ） を開きます。
2. キーボードで Command (⌘) + Shift + .（ピリオド） を押します。
3. うっすら半透明になった隠しフォルダ（usr や opt など）が表示されるようになります。
4. あとは usr ➔ local ➔ var ➔ www とダブルクリックで順にたどっていけばアクセスできます。


/usr/local/などのファイルパスで
> open .
とすると、finderが開けるが、
> code .
でVSCodeを開けるようにしたい。

設定手順（最初の一回だけ）
1. VS Code を普通にアプリ一覧から起動します。
2. キーボードで Command (⌘) + Shift + P を押します（コマンドパレットが開きます）。
3. 入力欄に shell command と入力します。
4. 候補に出てくる 「Shell Command: Install 'code' command in PATH」 を選択してクリックします。 (日本語設定の場合：「シェル コマンド: PATH 内に 'code' コマンドをインストールします」)
5. 「成功しました」という通知（またはダイアログ）が出たら設定完了です！（VS Codeは閉じてOKです）


めも

> httpd -v
Apacheのバージョンだけ表示するコマンド


> httpd -V
Apacheが実際にどの設定ファイルを読んで起動しているかを教えてくれる

```
 -D HTTPD_ROOT="/usr/local/Cellar/httpd/2.4.68"
 -D SERVER_CONFIG_FILE="/usr/local/etc/httpd/httpd.conf"
```
