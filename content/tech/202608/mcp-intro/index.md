---
date: '2026-08-21T10:00:00+09:00'
draft: false
title: 'MCP超入門 ── AIに「道具」を持たせる仕組みを理解する'
categories: ["Tech"]
tags: ["MCP", "AI", "Claude", "LLM", "Claude Code"]
summary: "MCPとは何か、なぜ必要か、どう動くかを、社内でAI活用を始めたい人向けにわかりやすく整理した。"
---

## 背景

社内のAI活用は人によってやり方がバラバラで、全体の運用ルールも整っていない。AIの実務経験がない新規メンバーが同じプロジェクトに加わったことをきっかけに、仕組みの共通化を考え始めた。

「MCPについて一から説明するとしたら？」という問いに答える形でまとめた記事。MCPを理解すると「AIが自分でファイルを読み、GitHubを操作し、データベースを参照する」レベルの自動化が現実的になる。MCPを知らない人が「なるほど、そういう仕組みか」と感じられることをゴールとする。

---

## そもそも何が問題だったか

通常のAIチャット（ChatGPT・Claudeなど）は、**会話の中にある情報しか見えない**。

```text
ユーザー: このコードのバグを直して
AI: （コードを貼ってもらわないと何も見えない）
```

実際の業務では

- リポジトリのコードを読んでほしい
- Slackのスレッドを確認してほしい
- データベースの値を参照してほしい
- ファイルを直接編集してほしい

といったことが頻繁にある。これらは「AIに手渡す」のが大変で、結局人間がコピペして橋渡しをしていた。

---

## AIが外部にアクセスする方法

AIが「会話の外にある情報」にアクセスする方法は、実はMCPだけではない。

| 方法 | 例 | イメージ |
|------|-----|---------|
| **MCP** | Figma MCP、GitHub MCP | 専用の窓口を通して操作 |
| **CLI** | `git`、`gh`、`aws` | シェルでコマンドを実行 |
| **API** | Twilio REST API | HTTPで直接呼ぶ |
| **ローカルファイル** | ソースコード、JSON | ファイルを直接読む |

たとえば GitHub の Issue を取得したい場合、MCPを使わなくても `gh issue list` コマンドをAIが実行することで同じことができる。

では**MCPの何がうれしいのか**というのが本題になる。

---

## MCPとは何か

**MCP（Model Context Protocol）**は、Anthropicが2024年に公開した、AIと外部ツールをつなぐための**標準的な接続規格（プロトコル）**。

ATMを思い浮かべるとイメージしやすい。

どの銀行のカードでも同じATMで引き出せるのは、銀行とATMが共通のプロトコルで通信しているから。異なるサービス同士でも「規格」さえ合っていればつながれる。

MCPはその「規格」のAI版。AIがどのツールに対しても同じ方法でアクセスできるよう、共通の接続ルールを定めたのがMCPだ。

重要なのは「MCPはAPIの代わり」ではなく、**「APIやローカル機能などをAIが扱いやすい共通形式で公開する層」**だという点。MCP Server の裏側では、結局その先のサービスのAPIを呼んでいることも多い。

```text
Claude Code
      ↓ MCP
Figma MCP Server
      ↓
Figma API  ← MCP Serverがここを叩いている
      ↓
Figma
```

### CLIと比べたときのMCPの強み

CLIの場合、AIは自分で「このサービスはどのコマンドを使えばいいのか」を考える必要がある。

MCPの場合、MCP Server側が次の情報をAIに提示する。

```text
利用できるツール:
  - search_code(query: string)  ← コードを検索する
  - get_file(path: string)      ← ファイルを取得する
  - create_pr(title, body)      ← PRを作成する
```

AIはこのツール一覧を受け取り、どのツールを呼べばいいかを判断できる。つまりMCPは**「AIと外部ツールのあいだに共通の会話形式を作った仕組み」**といえる。

---

## MCPの構成要素

MCPには3つの登場人物がいる。

| 役割 | 説明 | 例 |
|------|------|----|
| **Host** | AIを動かすアプリケーション | Claude Code |
| **Client** | HostとServerをつなぐ仲介役 | Hostに内蔵されている |
| **Server** | 外部ツールやデータを提供する小さなプログラム | Figma MCP Server、Redmine MCP Server |

```text
ユーザー
   │
   ▼
Host（Claude Code）
   │
   ▼（MCP経由）
MCP Server（GitHub / ファイルシステム / DB など）
   │
   ▼
実際のツール・データ
```

---

## MCPサーバーは誰が作るのか

MCPサーバーは、大きく3つのパターンがある。

**1. サービス提供企業が公式に作る**

FigmaやNotionなど、企業が自社サービスのMCPサーバーを公式提供している。

```text
Claude Code → Figma MCP Server（Figma公式）→ Figma
```

**2. OSSコミュニティや第三者が作る**

公式サーバーがない場合でも、コミュニティがMCPサーバーを作って公開していることがある。

**3. 自分たちで作る**

社内システムや、MCPサーバーが存在しないサービスに接続したい場合、自作できる。

```text
Claude Code → 自作 Redmine MCP Server → 社内Redmine API → Redmine
```

つまりMCPサーバーは「FigmaやGitHubそのもの」ではなく、**「AIからそのサービスを操作するための橋渡しプログラム」**だと考えるとわかりやすい。

---

## MCPサーバーで何ができるか

MCPサーバーはすでに多数が公開されており、設定するだけで使える。代表的なものをいくつか挙げる。

| MCPサーバー | できること |
|------------|-----------|
| **Figma** | デザインデータ・コンポーネント情報の参照 |
| **Lychee** | チケットの参照・更新、進捗確認 |
| **Redmine** | チケット検索・登録・ステータス更新 |
| **CodeGraph** | コードの依存関係・呼び出し元の検索 |
| **GitHub** | Issue・PRの読み書き、コード検索 |
| **Slack** | チャンネル・スレッドの参照、投稿 |

これらをClaude Codeに設定しておくと、AIが自律的に情報を取りに行けるようになる。

---

## 動作のイメージ

たとえば「このチケットの内容を実装して」という依頼をした場合。

**MCPなし:**
```text
ユーザー: このチケットを実装して（チケット内容をコピペ、関連コードもコピペ）
AI: ここを修正すればいいと思います（提案のみ）
ユーザー: （手動でファイルを書き換える）
```

**MCPあり（Redmine + CodeGraph）:**
```text
ユーザー: チケット #456 を実装してください
AI: （Redmine MCPでチケット内容を取得）
AI: （CodeGraph MCPで関連コードの依存関係を調査）
AI: （ファイルを直接書き換える ※Claude Code組み込み機能）
AI: 実装しました。チケット #456 の要件に対応しています
```

人間がやっていた「チケット確認」「コード調査」「ファイル編集」をAIが代行する。

---

## MCPサーバーを設定してみる

ターミナルで `claude mcp add` コマンドを使って追加する。`--scope` オプションで設定の有効範囲を指定できる。

### スコープの使い分け

| スコープ | 書き込まれるファイル | 有効範囲 |
|---------|------------------|---------|
| `local`（デフォルト） | `~/.claude.json`（プロジェクト紐付き） | 自分・このプロジェクトのみ |
| `project` | プロジェクトルートの `.mcp.json` | チーム全員・このプロジェクトのみ（Git管理） |
| `user` | `~/.claude.json` | 自分・全プロジェクト共通 |

複数のプロジェクトを横断して使うサーバーは `user` スコープが適している。たとえばdev配下にA・B・Cプロジェクトがあり、FigmaやRedmineのMCPをどのプロジェクトでも使いたい場合は、`user` スコープで1回登録するだけでよい。

```bash
# 全プロジェクト共通で使うもの → userスコープ
claude mcp add figma --scope user -- npx -y figma-mcp
claude mcp add redmine --scope user -- npx -y redmine-mcp
```

特定プロジェクト専用のサーバー（そのリポジトリ固有のDBなど）をチームで共有したい場合は `project` スコープを使う。`.mcp.json` がGitにコミットされるので、チームメンバーが `git pull` するだけで同じ設定が使えるようになる。

```bash
# チームで共有する必要があるもの → projectスコープ
claude mcp add knowledge-server --scope project -- npx -y knowledge-mcp
```

同じ名前のサーバーが複数のスコープに存在する場合、`.mcp.json`（project）が `user` より優先される。`user` に登録済みのサーバーが特定プロジェクトの `.mcp.json` にも重複して存在しても壊れはしないが、冗長になる。横断利用が目的なら `user` スコープに一本化して、`.mcp.json` には本当にそのプロジェクト固有のものだけ置くのが整理しやすい。

設定後、`claude mcp list` で追加されたサーバーを確認できる。

---

## 自社システム用のMCPサーバーを作る

公開されているサーバー以外に、自社システムに接続するMCPサーバーを自分で作ることもできる。

Node.jsまたはPythonで、以下のような構成で書く。

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";

const server = new Server({ name: "my-server", version: "1.0.0" });

// AIが呼び出せる「ツール」を定義する
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_ticket",
      description: "チケットIDから社内Redmineのチケット情報を取得する",
      inputSchema: {
        type: "object",
        properties: {
          ticketId: { type: "string" }
        }
      }
    }
  ]
}));

// ツールが呼ばれたときの処理
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_ticket") {
    const ticket = await redmineApi.getTicket(request.params.arguments.ticketId);
    return { content: [{ type: "text", text: JSON.stringify(ticket) }] };
  }
});
```

これを設定すると、「このチケットの内容をまとめて実装してください」といった依頼をAIに投げることができる。

---

## よくある疑問

**Q. セキュリティは大丈夫？**

MCPサーバーはローカルで動かすことが基本なので、外部にデータが送られるわけではない。AIに渡す情報の範囲はMCPサーバーの設定で制御できる。ただしAPIキーの管理など基本的なセキュリティ対策は必要。

**Q. プログラミングの知識がないと使えない？**

設定済みのMCPサーバーを追加するだけなら、JSONファイルを編集できれば問題ない。自作サーバーを作る場合はコーディングが必要になる。

**Q. ChatGPTでも使える？**

現時点ではClaude（Anthropic製）やClaude Code、そして一部のサードパーティツールがMCPに対応している。OpenAIも対応を発表しており、今後は複数のAIツールでMCPが使えるようになる見込み。

**Q. MCPを使えばAIはなんでもできる？**

MCPはあくまで「AIと外部ツールをつなぐ規格」であり、接続できるのはMCPサーバーとして実装された機能だけ。どんな操作でもできるわけではなく、何ができるかはMCPサーバーの設計次第。

---

## まとめ

| 項目 | 内容 |
|------|------|
| MCPとは | AIと外部ツールをつなぐ標準規格 |
| なぜ必要か | CLIやAPIより「AIが使いやすい共通インターフェース」を提供するため |
| 何ができるか | ファイル操作・GitHub・DB・SlackなどをAIが自律的に使える |
| MCPサーバーは誰が作る？ | サービス企業・OSSコミュニティ・自社、いずれも可 |
| 始め方 | `claude mcp add` コマンドでMCPサーバーを追加する |
| 発展 | 社内システム向けのMCPサーバーを自作できる |

MCPを活用すると、「AIに相談して自分で作業する」から「AIに依頼して任せる」という働き方にシフトできる。

まずは `claude mcp add` でFigmaやRedmineのMCPサーバーを追加して、AIに情報を取りに行かせるところから試してみるのがおすすめ。
