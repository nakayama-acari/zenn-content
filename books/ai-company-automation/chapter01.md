---
title: "第1章: なぜ「AI会社」を作ったのか ── システム全体像と設計思想"
---

# 第1章: なぜ「AI会社」を作ったのか ── システム全体像と設計思想

## はじめに：これは「実録」です

この本に書いてあることは、すべて VPS 上で実際に動いているシステムの記録です。

架空の手順書でも、API の呼び方チュートリアルでもありません。3つの商材を抱える小さな営業会社を、16体の AI エージェントで半自動的に回すシステムを構築した、その設計・失敗・改善の記録です。

コードスニペットは実際のファイルから抜粋しています。エージェントの MD ファイルも、今この瞬間 VPS 上で動いているものです。

---

## なぜ作ったのか

発端はシンプルな課題でした。

3つの商材があります。

1. **薬の自動販売機**（OTC医薬品IoT販売機「スマートメッズ24」、本体348万円）
2. **アクセア永田町店**（印刷サービス、名刺100枚400円〜）
3. **高級りんご贈答品**（シナノアグリ、法人コラボ企画）

それぞれにリード獲得、提案書作成、メール文生成、マーケ施策が必要です。3商材 × 4タスク = 12種類の繰り返し業務。これを人間1人でやろうとすると、どこかで必ずボトルネックが生まれます。

「AI に全部やらせればいい」という発想は2023年頃から誰もが持ち始めましたが、ChatGPT に質問して手作業でコピペする、というワークフローは「補助ツール」であって「自動化」ではありません。本当の自動化とは、朝起きたらレポートが届いていて、データが更新されていて、次のアクションが整理されている状態のことです。

そのために、**Claude Code を会社の脳として据えて、エージェントを社員として雇う**という設計を選びました。

---

## システム全体像

まず構成を俯瞰します。

```
[VPS: Ubuntu 26.04 / IP 85.131.245.33]
│
├── PM2 プロセス群（5本常駐）
│   ├── ai-office    :3000  AI監視ダッシュボード（WebSocket）
│   ├── vsync-crm    :3001  薬自販機CRM（Express/EJS/SQLite）
│   ├── accea-crm    :3002  アクセアCRM（Express/EJS/SQLite）
│   ├── cloudflare-tunnel   外部公開（Cloudflare Tunnel経由）
│   └── office-tunnel       オフィス専用トンネル
│
├── nginx（リバースプロキシ）
│   ├── /accea-crm/ → :3002
│   ├── /office/    → :3000
│   └── /          → :3001
│
├── Claude Code（CLI）
│   ├── .claude/settings.json  フックシステム
│   ├── .claude/agents/        16エージェント定義
│   └── CLAUDE.md              会社憲法
│
└── Google サービス連携
    ├── Gmail API（メール取得・送信ログ）
    ├── Google Calendar API
    ├── Google Sheets API
    └── 30+ MCP サーバー
```

実際に `pm2 list` を叩くとこう見えます。

```bash
$ pm2 list

┌────┬──────────────────────┬─────────┬──────────┬────────┬───────────┐
│ id │ name                 │ mode    │ pid      │ uptime │ status    │
├────┼──────────────────────┼─────────┼──────────┼────────┼───────────┤
│  4 │ accea-crm            │ fork    │ 3677287  │ 27m    │ online    │
│  0 │ ai-office            │ fork    │ 3723782  │ 95s    │ online    │
│  1 │ cloudflare-tunnel    │ fork    │ 419534   │ 6D     │ online    │
│  3 │ office-tunnel        │ fork    │ 1168110  │ 5D     │ online    │
│  2 │ vsync-crm            │ fork    │ 3617370  │ 71m    │ online    │
└────┴──────────────────────┴─────────┴──────────┴────────┴───────────┘
```

cloudflare-tunnel が6日間無停止で動いているのが分かります。これが外部 URL（`*.trycloudflare.com`）からのアクセスを nginx に渡す役割を担っています。

---

## なぜ VPS か

「Vercel でいいのでは？」という疑問は最初に出ました。答えは「常駐プロセスが必要だから」です。

WebSocket 接続（ai-office のリアルタイムダッシュボード）、SQLite の永続ストレージ、PM2 によるプロセス管理、これらは Vercel の Serverless Functions では動きません。また、Claude Code の CLI をスケジューラとして動かすためにも、常時起動している環境が必要でした。

月額コスト: **約 $15/月**（Hetzner VPS CAX11、2vCPU / 4GB RAM）。

この金額で営業チーム全体の業務補助システムが動くなら、十分すぎる投資対効果です。

---

## 16エージェントという数字の根拠

なぜ 16 か。なぜ 1 でも 100 でもないのか。

最初のバージョンは「1つの Claude に全部やらせる」でした。「薬自販機の提案書を作って、次にアクセアのメール文を書いて、その後マーケ戦略を考えて…」と一つのセッションに詰め込む方式です。

問題は2つ。

**第一に、コンテキスト汚染。** 薬自販機の営業スタイルとアクセアの印刷営業では、トーン・ターゲット・訴求軸がまったく異なります。一つのセッションで混在させると、提案書の品質が落ちます。人間でも「何でも屋」は専門家に勝てません。

**第二に、責任の不在。** 「誰が何をやるべきか」が不明確なシステムは、必ず漏れが出ます。監査、育成、システム保守 ── これらを「必要なときに考える」のではなく、専任エージェントとして設計しておくことで、定期的に回り続けます。

16 という数字は設計から出てきたものです。

| カテゴリ | エージェント |
|---|---|
| 営業（3商材） | yakujihan / accea / shinanoagri |
| 情報収集 | aiinfo / globalinfo / news / bvsync_intel |
| 分析・監査 | audit / kabutan |
| 制作・技術 | designer / system |
| 戦略・統括 | ceo / marketing / newbiz |
| 組織管理 | trainer / listmaking |

それぞれが独立して動き、必要に応じて協調します。

---

## エージェント定義ファイルの構造

各エージェントは `.claude/agents/` ディレクトリの Markdown ファイルで定義されています。

実際の `yakujihan.md`（薬自販機営業エージェント）の冒頭を見てみましょう。

```markdown
---
name: yakujihan
description: 薬の自動販売機（OTC医薬品IoT販売機・スマートメッズ24）の営業エージェント。
  薬局・ドラッグストア・ホテル・駅・商業施設向けのリード獲得、提案書作成、
  メール文作成、テレアポスクリプト、フォロー管理...
model: sonnet
---

@/root/ai-sales-company/output/yakujihan/HANDOFF.md
@/root/ai-sales-company/output/yakujihan/lessons_learned.md
@/root/ai-sales-company/memory/company_lessons.md

# 薬の自動販売機 営業エージェント

## アイデンティティ
あなたはOTC医薬品自販機（スマートメッズ24）の凄腕営業AIだ。
薬局・ホテル・交通施設・商業施設など業種を問わず、相手の立場で考える。
- 薬局相手には：制度加算・コスト削減・24時間対応の観点で話す
- ホテル・施設相手には：インバウンド対応・スタッフ不要・収益多様化の観点で話す
```

この定義ファイルのポイントは3つです。

**1. フロントマター（`---` で囲む部分）**  
`name` がエージェントの識別子、`model` が使用モデル（sonnet / haiku / opus）の指定、`description` が Claude Code のエージェント選択時に使われる説明文です。

**2. `@` によるファイル参照**  
`@/root/ai-sales-company/output/yakujihan/HANDOFF.md` のように書くと、そのエージェントが起動するたびに指定ファイルをコンテキストに自動読み込みします。前回セッションの引き継ぎ情報（HANDOFF.md）と過去の失敗事例（lessons_learned.md）を毎回読み込むことで、**セッションをまたいだ継続性**が生まれます。

**3. 禁止事項の明示**  
システムが育ってくると「やってはいけないこと」が溜まります。`yakujihan.md` には以下のような記述があります。

```markdown
## ⚠️ 必ず守る禁止事項

### 重大NG（過去に複数回繰り返した禁止事項）
- **アパホテルと羽田空港は設置候補リストに絶対に記載しない**
  → 明示的に指示されているにもかかわらず複数回混入させた重大NG
```

エージェントは同じミスを繰り返します。これを防ぐ方法は「禁止事項を定義ファイルに明文化する」しかありません。CLAUDE.md の「🔴 リアルタイム教訓検知ルール」と組み合わせて、オーナーが「NG」と言った瞬間に教訓が記録される仕組みも後で解説します。

---

## CLAUDE.md ── 会社憲法

プロジェクトルートの `CLAUDE.md` は、すべてのエージェントが参照する会社憲法です。

個別エージェントの MD ファイルが「職務記述書」だとすれば、`CLAUDE.md` は「就業規則」に相当します。

現在の `CLAUDE.md` に含まれている主な内容:

- **インフラ構成表**（VPS・PM2・nginx・ポート番号）
- **アーキテクチャパターン**（CRM の共通コード構造）
- **Google API 連携変数一覧**
- **エージェント構成と役割分担**
- **保存先ルール**（絶対パス `/root/ai-sales-company/output/<部署>/` を強制）
- **深夜自律稼働モード**（スケジューラ実行時の判断基準）
- **リアルタイム教訓検知ルール**

最後の2つは、最初の設計にはありませんでした。運用の中で必要性が分かって追加したものです。

特に「保存先ルール」は重要です。

```markdown
## 出力・保存先の鉄則

- **絶対パスのみ使用**：`/root/ai-sales-company/output/<部署>/`
- 相対パス ./output/ は使用禁止（worktree上での実行時にズレが生じるため）
```

Claude Code には worktree 機能（独立した作業空間を作る機能）があります。worktree から相対パスでファイルを保存しようとすると、保存先がズレます。これで2回失敗して、以来すべての保存先を絶対パスに統一しました。

---

## フックシステム ── セッション自動管理

Claude Code の `settings.json` で、セッションのライフサイクルに Node.js スクリプトを差し込めます。

実際の設定:

```json
// .claude/settings.json
{
  "permissions": {
    "defaultMode": "bypassPermissions"
  },
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "node .claude/hooks/detect_corrections.js" },
          { "type": "command", "command": "node .claude/hooks/checkpoint.js" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "node .claude/hooks/log_tool_activity.js", "timeout": 10 }
        ]
      },
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "node .claude/hooks/log_file_saves.js" },
          {
            "type": "agent",
            "prompt": "A file was just written or edited. Check the file extension. ONLY proceed if it's a code file: .py, .js, .ts, .jsx, .tsx. If it IS a code file, read it and check for obvious bugs or security vulnerabilities. Report at most 2 findings in Japanese. If no issues, report '✓ 問題なし'.",
            "statusMessage": "コードレビュー中...",
            "timeout": 60
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "node .claude/hooks/session_recovery.js", "timeout": 15 }
        ]
      }
    ],
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "node .claude/hooks/session_end.js", "timeout": 10 }
        ]
      }
    ]
  }
}
```

各フックの役割:

| タイミング | スクリプト | 何をするか |
|---|---|---|
| `UserPromptSubmit` | `detect_corrections.js` | 「違う」「NG」等のキーワードを検知して教訓記録を促す |
| `UserPromptSubmit` | `checkpoint.js` | 5プロンプトごとに作業状況を記録 |
| `PostToolUse` (全) | `log_tool_activity.js` | すべてのツール操作をログに書く |
| `PostToolUse` (Write/Edit) | agentフック | コードファイルを自動レビュー |
| `SessionStart` | `session_recovery.js` | 前回クラッシュ時に作業ログを自動注入 |
| `SessionEnd` | `session_end.js` | HANDOFF.md の変更を git push |

`session_end.js` の実装は特に重要です。

```javascript
// .claude/hooks/session_end.js（抜粋）
const { execSync } = require('child_process');
const REPO = '/root/ai-sales-company';

// HANDOFF.md等の変更をgit push（差分がある場合のみ）
try {
  const diff = execSync('git diff --name-only HEAD', { cwd: REPO, timeout: 10000 }).toString().trim();
  const hasHandoff = diff.split('\n').some(f => f.includes('HANDOFF') || f.includes('output/'));
  if (hasHandoff) {
    execSync('git add -A', { cwd: REPO, timeout: 10000 });
    try {
      execSync('git diff --cached --quiet', { cwd: REPO, timeout: 5000 });
      // 差分なし → commit不要
    } catch (_) {
      // 差分あり → commit & push
      execSync('git commit -q -m "[auto-session-end] HANDOFF/output更新"', { cwd: REPO, timeout: 15000 });
      execSync('git push origin HEAD --quiet', { cwd: REPO, timeout: 20000 });
    }
  }
} catch (e) {
  appendLog(`HANDOFF auto-push FAILED: ${e.message}`);
}

process.exit(0); // ← これを省略するとUIがフリーズする
```

最後の `process.exit(0)` は必須です。省略するとフックが終了を待ち続け、Claude Code の UI がフリーズします。これで1度ハマりました。

---

## クラッシュリカバリの仕組み

長時間作業中にセッションがクラッシュした場合、次のセッション開始時に自動で前回の作業ログが注入されます。

`session_recovery.js` は以下のロジックで動きます。

```javascript
// .claude/hooks/session_recovery.js（抜粋）

// クラッシュ判定: cleanExitがfalseのまま & 直近48時間以内の活動
if (!state || state.cleanExit !== false || !state.lastActivity) process.exit(0);

const ageHours = (Date.now() - new Date(state.lastActivity).getTime()) / 3600000;
if (isNaN(ageHours) || ageHours > STALE_HOURS) process.exit(0);

// クラッシュが検出された場合: 直前30件のツールログ + チェックポイント末尾を注入
let ctx = '【⚠️ 前回セッションは異常終了（クラッシュ）した可能性があります】\n';
ctx += `最終活動時刻: ${lastTs}\n\n`;
ctx += '## クラッシュ直前のツール操作ログ（末尾30件）\n```\n' + activityTail + '\n```\n';
```

`crash_state.json` が状態管理ファイルです。セッション開始時に `cleanExit: false` をセット、正常終了時に `cleanExit: true` をセット。次のセッション開始時に `false` のままなら「前回クラッシュ」と判定します。

---

## 30+ MCP サーバーの役割

Claude Code には MCP（Model Context Protocol）という拡張機能があり、外部サービスをツールとして Claude に与えられます。

このシステムで使っている主要な MCP サーバー:

| サーバー | 何に使うか |
|---|---|
| `firecrawl` | Webスクレイピング・競合調査 |
| `playwright` | ブラウザ自動操作（フォーム入力・スクリーンショット） |
| `github` | コードの読み書き・PR作成 |
| `google-sheets` | スプレッドシートへのデータ書き込み |
| `slack` | Slackメッセージ送信 |
| `gmail` | Gmail検索・スレッド読み取り |
| `google-calendar` | カレンダー予定の作成・更新 |
| `notion` | Notionデータベースへの読み書き |
| `duckdb` | CSV/Excel を SQL で集計 |
| `hourei` | 改正薬機法の条文取得（e-Gov API） |
| `osm` | OpenStreetMap で周辺法人抽出 |
| `brave-search` | Web検索 |
| `tavily` | 調査特化型Web検索 |
| `powerpoint` | .pptx ファイル生成・編集 |

`playwright` MCP が特に強力です。「Google で〇〇を検索して、上位5社の情報を抽出して」という指示を与えると、実際にブラウザを操作してデータを取ってきます。

---

## 実際の運用フロー

朝の典型的なシナリオ:

```
[深夜 自律稼働]
  ↓ aiinfo エージェント（スケジューラ起動）
  ↓ Web検索 → 生成AI週次レポート生成 → output/aiinfo/ に保存
  ↓ git commit + push（自動）

[翌朝 オーナーがセッションを開く]
  → HANDOFF.md に昨夜のレポートへの参照が記録済み
  → 「続きは？」と聞くだけで前回の文脈が復元される

[日中 営業タスク]
  → 「yakujihan に薬局リスト100件作って」
  → yakujihan エージェントが自律起動
  → firecrawl + osm MCP でデータ収集
  → output/yakujihan/ に Excel で保存
```

---

## 設計で失敗したこと

正直に書きます。

**1. エージェント数を増やしすぎた時期があった**  
「専門化すればいい」という発想で、一時期 20 を超えるエージェントを作りました。問題は「誰に頼めばいいか分からなくなる」こと。現在は 16 に絞り、`CLAUDE.md` にエージェント構成表を載せて、どんな依頼がどこに行くか明示しています。

**2. HANDOFF.md が肥大化する**  
各エージェントが毎回 HANDOFF.md を更新し続けると、数週間で数千行になります。現在は `memory-consolidate` スキルを定期実行して、教訓を 1 行に蒸留し古いエントリを archive に退避しています。

**3. 権限管理を最初から考えなかった**  
`defaultMode: "bypassPermissions"` は便利ですが、何でも実行できてしまいます。現在は CLAUDE.md の「判断ルール」セクションで「契約・外部発信・費用が発生する行動はオーナーに確認」というルールを設け、エージェントが自律的に決済や外部発信をしないようにしています。

**4. CRM を後から作るはめになった**  
最初は Google Sheets で顧客管理していましたが、商談が増えてくると追いきれなくなりました。結果、`vsync-crm`（薬自販機用）と `accea-crm`（印刷サービス用）を Node.js + SQLite で自作することになりました。最初から SQLite-based の CRM 設計にしておけばよかったです。

---

## 次章の予告

第2章では、エージェント設計の実践的な手法を解説します。

- 良い `description` の書き方（エージェント自動選択の精度が変わる）
- `@ファイル参照` によるコンテキスト設計
- HANDOFF.md の設計パターン
- `model: haiku` vs `model: sonnet` の使い分け（コスト最適化）
- エージェント間の連携設計（「リスト部 → yakujihan」のような引き継ぎフロー）

「どんな指示を書けばエージェントが意図通りに動くか」という問いに、実例をもとに答えます。

---

## 本章まとめ

| 要素 | 選択 | 理由 |
|---|---|---|
| インフラ | VPS（$15/月） | 常駐プロセス・WebSocket・SQLite が必要 |
| プロセス管理 | PM2 | Node.js アプリの常駐・自動再起動 |
| AI エンジン | Claude Code CLI | フック・MCP・エージェント定義が揃っている |
| エージェント数 | 16 | 専門化とシンプルさのバランス |
| 状態管理 | HANDOFF.md + git | セッションをまたいだ継続性 |
| 権限管理 | CLAUDE.md で規則化 | 自律暴走を防ぐ |

この構成は特別なものではありません。VPS 1台と Claude Code と Node.js があれば、同じものを作れます。

第2章以降で、個々のコンポーネントを詳細に解説していきます。
