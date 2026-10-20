# only-claude-window

[Flue](https://github.com/withastro/flue)（Astroチームのエージェントフレームワーク、v2.3.0時点）を使って、
Claudeのウィンドウ1枚で Notion・Slack・md・コード閲覧までの作業を完結させる環境の構成案。

## 結論

- Flueが担えるのは**裏側（バックエンド）**。Claudeを動かし、md・コード・Notion・Slack・定期実行・会話の保存を1つのエージェントにまとめられる。
- **画面（ウィンドウ）は付いてこない**。Flueは画面を持たないフレームワークで、チャット画面は `@flue/react` で自作する（公式のデモ画面を土台にできる）。
- IDEのようなコード表示機能もないため、自作するか既存のもの（code-server など）を埋め込む。

## やりたいこと別：Flueでどこまでできるか

| やりたいこと | Flueでの実現方法 | 判定 | 補足 |
|---|---|---|---|
| Claudeで作業する | `useModel('anthropic/claude-sonnet-5-5')` | ✅ | **APIキーの従量課金**が前提。Pro/Maxのサブスク枠では使えない |
| mdを読み書きする | `useSandbox(local())` を付けると、read / write / edit / grep / glob / bash が使える | ✅ | 自分のPCのフォルダをそのまま作業場所にできる |
| コードを読む（AI側） | 同上＋git clone。リポジトリ単位でサブエージェントに任せる | ✅ | 作業フォルダの `AGENTS.md` と `.agents/skills/` を自動で読み込む |
| **IDEのような画面** | **機能なし** | ❌ 自作 | ファイルツリーとハイライト表示はReactで作る。本格的なIDEが必要なら code-server（ブラウザ版VS Code）を埋め込む |
| Notionの読み書き | `@notionhq/client` をツールにする、または `useMcpConnection` でNotion MCPにつなぐ | ✅ | MCPのOAuthは自分で実装する必要がある（Flueはトークンを管理しない）。自分専用ならインテグレーショントークン＋ツールが簡単 |
| Notionの更新をきっかけに動かす | `@flue/notion`（Webhookの受信） | ✅ | 外から届くURLが必要（ローカルならトンネルを使う） |
| Slackを読む・投稿する | `@slack/web-api` をツールにする | ✅ | **Flueには送信機能がない**ので送信は自分のコードで書く。メッセージ検索にはユーザートークンが必要 |
| Slackのメンションで起動する | `@flue/slack` | ✅ | 公開URLが必要 |
| 定期実行（朝のまとめなど） | cron（croner）＋`dispatch()` | ✅ | Flue自体はスケジューラを持たず、cronから呼び出す |
| 送信前に人が承認する | 公式ドキュメントの「Approval gate」パターン | ⚠️ | 組み込みAPIではなく、ドキュメントの例をコピーして使う |
| 会話の保存・再開 | Durable streams（耐久ログ）＋sqlite/Postgres | ✅ | 再起動しても続きから再開できる |
| チャット内にカードを出す | `useDataWriter` で送り、React側で描画 | ✅ | Notion下書き・差分・ファイルのプレビューを表示できる |
| PDFや画像を渡す | `sendMessage(..., { documents, images })` | ✅ | |

## 構成案

```
[ブラウザ：1枚のウィンドウ]  Vite + React + @flue/react（公式のデモ画面を改造）
   │  チャット   : HTTP + SSE（@flue/sdk）
   │  右側の表示 : /api/workspace/*（ファイルツリーと中身。AIを通さずに直接読む）
   ▼
[Flueアプリ]  Node.js / Hono（src/app.ts） ← 自分のPC、または常時起動のVM
   ├─ agents/workbench.ts    メインのエージェント
   │    ├─ useSandbox(local())    → 作業フォルダの md / repos を読む・編集・検索・実行
   │    ├─ useTool(notion_*)      → @notionhq/client
   │    ├─ useTool(slack_*)       → @slack/web-api（書き込みは承認を通す）
   │    ├─ useMcpConnection(...)  → GitHub / Linear などMCPがあるサービス
   │    ├─ useSkill(...)          → 朝のまとめやPRレビューの手順書（SKILL.md）
   │    ├─ useSubagent(...)       → code-reader（リポジトリ単位で調査）
   │    └─ useDataWriter(...)     → 画面にカード（Notion下書き・差分・ファイル）を出す
   ├─ channels/slack.ts      @メンションで同じエージェントを起動
   ├─ channels/notion.ts     ページ更新でエージェントを起動
   ├─ cron（croner）         平日8:45に朝のまとめを開始
   └─ db: sqlite             会話と状態を保存
        ▼
  Anthropic API / Notion API / Slack Web API / GitHub / ~/workspace（repos/, notes/*.md）
```

**動かす場所はNode.jsでローカル実行（`local()`）がおすすめ。** 手元のmdやリポジトリを直接扱える。
Cloudflareでも動かせるが、その場合はリモートの実行環境が必要になり、ローカルのファイルには届かない。

### ディレクトリ構成（予定）

```
only-claude-window/
├─ vite.config.ts            # flue() プラグイン
├─ src/
│  ├─ app.ts                 # エージェントのルート、チャネル、workspace API、cron
│  ├─ agents/workbench.ts    # メインのエージェント
│  ├─ tools/
│  │  ├─ notion.ts           # 検索・取得・作成・更新
│  │  ├─ slack.ts            # スレッド取得・検索・投稿（承認付き）
│  │  └─ approval.ts         # 承認ゲート（公式パターン）
│  ├─ channels/              # slack.ts / notion.ts（Webhook受信）
│  ├─ skills/                # daily-brief/SKILL.md など
│  └─ ui/                    # React（デモ画面を流用）
│     ├─ App.tsx             # 3列レイアウト
│     ├─ panes/              # CodeViewer / MarkdownPreview
│     └─ cards/              # NotionPageCard / SlackThreadCard / DiffCard
└─ workspace/                # repos/, notes/*.md, AGENTS.md, .agents/skills/
```

### メインのエージェント（イメージ）

```ts
// src/agents/workbench.ts
'use agent';
import { useModel, useSandbox, useSkill, useSubagent, useTool } from '@flue/runtime';
import { local } from '@flue/runtime/node';
import { notionTools } from '../tools/notion.ts';
import { slackTools } from '../tools/slack.ts';
import dailyBrief from '../skills/daily-brief/SKILL.md';

export function Workbench() {
  useModel('anthropic/claude-sonnet-5-5', {
    compaction: { model: 'anthropic/claude-haiku-5-5' }, // 長い会話の要約は安いモデルで
  });
  useSandbox(local({ cwd: process.env.WORKSPACE_DIR, env: { GH_TOKEN: process.env.GH_TOKEN } }));
  for (const tool of [...notionTools, ...slackTools]) useTool(tool);
  useSkill(dailyBrief);
  useSubagent({
    name: 'code-reader',
    description: 'リポジトリを読んで原因や構造を調べる',
    agent: () => '根拠となるファイルパスと行番号を付けて報告する。',
  });
  return 'あなたは私専用の作業台。Slack・Notion・ローカルのmdとコードを横断して作業する。外部への書き込みは必ず承認を取る。';
}
Workbench.agentName = 'workbench';
```

```ts
// src/app.ts
const app = new Hono();
app.route('/api/agents/workbench', createAgentRouter(Workbench)); // チャット
app.route('/api/workspace', workspaceApi);                         // 右側のコード・md表示
app.route('/channels/slack', slack.route());                        // Slackメンションの受信
new Cron('45 8 * * 1-5', { timezone: 'Asia/Tokyo' }, () =>
  dispatch(Workbench, { id: `brief-${today()}`, message: { kind: 'signal', type: 'schedule', body: '朝のまとめを作成' } }));
```

## 完成イメージ

```
+-- Only Claude Window -------------------------------------------------------------------+
| 会話一覧             | チャット                              | コンテキスト               |
|----------------------+---------------------------------------+----------------------------|
| * 朝のブリーフ 9:00  | あなた> #dev の昨日のスレのバグを     | [Code] MD  Notion  Slack   |
| * #dev バグ調査      |         調べて障害DBにまとめて        | repos/app/src/auth/        |
|   PR#42 レビュー     |                                       |  session.ts                |
|   週報ドラフト       | Claude> > slack_read_thread #dev      | -------------------------- |
|                      |         > grep 'expires' repos/app    | 40 const now = Date.now()  |
|                      |         > read src/auth/session.ts    | 41 if (exp < now) {        |
|                      |   原因は秒とミリ秒の比較ミスです      | 42   return null   <- 原因 |
|                      |   (session.ts 42行目 -> 右に表示)     | 43 }                       |
|                      |   +- Notion 下書き ----------------+  |                            |
|                      |   | 障害: ログイン直後に切れる     |  |                            |
|                      |   | 原因 / 影響範囲 / 修正案       |  |                            |
|                      |   | [承認して作成]  [修正を依頼]   |  |                            |
|                      |   +--------------------------------+  |                            |
|                      |   +- Slack 返信案 #dev ------------+  |                            |
|                      |   | [承認して送信]  [修正を依頼]   |  |                            |
|                      |   +--------------------------------+  |                            |
|                      |                                       |                            |
|                      | ------------------------------------- |                            |
| + 新しい会話         | > 入力 / PDF・画像をドロップ          | [VS Code で開く] [diff]    |
+-----------------------------------------------------------------------------------------+
```

### 1日の流れの例

1. 8:45に、Slackの未読・Notionのタスク・昨日のPRをまとめた会話が自動で作られている。
2. チャットで依頼すると、Slackを読み、コードを検索し、該当ファイルを右側に表示し、Notionの下書きをカードで出す。**承認**を押すと作成される。
3. 外出先からはSlackで `@bot` とメンションすれば、同じエージェントが動く。

## 注意点・限界

- **費用はAPIの従量課金**（`ANTHROPIC_API_KEY`）。
- `local()` は隔離された環境ではない。PCのファイルを直接触るので、作業フォルダを限定し、渡すトークンも最小限にする。
- OAuth、トークンの保管、画面のログイン認証は自分で作る。自分専用なら、localhostだけで待ち受ければ十分。
- 型チェックや定義ジャンプ、デバッガといった本物のIDE機能は作れない。必要ならcode-serverを埋め込むか、「VS Codeで開く」ボタンで外に逃がす。
- Flueはまだ若く（v2系）、変更が活発。`flue add channel slack` などのコマンドは、Claude Codeのようなコーディングエージェントに適用させる手順書（Markdown）を出す形式。

**作らずに済ませる方法との比較：** 「1つのClaudeウィンドウで仕事をする」こと自体は、Claudeアプリ＋Notion/Slackコネクタや、Claude CodeのIDE拡張でも7〜8割はできる。Flueを選ぶ価値があるのは次のような場合。

- 自分専用の3列の画面がほしい
- Slack・Notion・cronをきっかけに常駐で自動で動かしたい
- 承認ルールなどの挙動をコードで完全に決めたい

## 進め方（段階的に）

1. **最小構成（MVP）**：`flue init --target node --deploy` を実行し、デモ画面とworkbenchエージェントを用意する。これでmdとコードをチャットから扱えるようになる。
2. Notion・Slackを読むツールを追加する。
3. 右側のコード表示・mdプレビューと、チャット内のカードを作る。
4. 書き込みと承認、Slack/Notionからの受信、cronを追加する。

## 出典

- [withastro/flue（GitHub）](https://github.com/withastro/flue)
- [Flue 公式ドキュメント](https://flueframework.com/)
- [Flue — The Agent Harness Framework（Best of JS）](https://bestofjs.org/projects/flue)
- [Flue: the open framework for building AI agents（flaviocopes）](https://flaviocopes.com/flue-framework/)
- [Flue: Headless, Programmable AI Agent Framework（Better Stack）](https://betterstack.com/community/guides/ai/flue-framework.md)
