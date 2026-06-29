---
title: "第3章: Hookシステム完全解説 — Claude Codeに「習慣」を持たせた方法"
---

## この章で学ぶこと

- `.claude/hooks/` の全8ファイルが何をしているか（実際のパスとコード付き）
- セッション終了時に自動git pushを実現した仕組み（`session_end.js`）
- クラッシュから自動復帰する仕組み（`session_recovery.js`）
- 「違う」「ダメ」を検知して教訓記録を促す仕組み（`detect_corrections.js`）

## 実際のシステム構成

### Hookファイル一覧（実際の `/root/ai-sales-company/.claude/hooks/`）

```
checkpoint.js         ← ユーザー発言を全記録 / 5件ごとにAI要約を要求
session_end.js        ← セッション正常終了時にHANDOFF.mdをgit push
detect_corrections.js ← 「違う」「ダメ」を検知 → 教訓記録を促す
log_file_saves.js     ← ファイル保存のたびにツールログを記録
log_tool_activity.js  ← 全ツール呼び出しをtool_activity.logに記録
save_tool_memory.js   ← 重要操作をメモリファイルに保存
session_recovery.js   ← 前回クラッシュを検出 → 復元コンテキストを注入
crash_state.json      ← クラッシュ/正常終了フラグを記録（JSではなくデータ）
```

`.claude/settings.json` に `hooks` セクションを追加するだけでClaude Codeが自動実行する。

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node /root/ai-sales-company/.claude/hooks/checkpoint.js"
          },
          {
            "type": "command",
            "command": "node /root/ai-sales-company/.claude/hooks/detect_corrections.js"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "node /root/ai-sales-company/.claude/hooks/log_tool_activity.js"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node /root/ai-sales-company/.claude/hooks/session_end.js"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node /root/ai-sales-company/.claude/hooks/session_recovery.js"
          }
        ]
      }
    ]
  }
}
```

## なぜこの設計にしたか

最初はHookなしで運用していた。問題が2つあった。

**問題1: HANDOFF.mdの更新忘れ**

AIは毎回「HANDOFF.mdを更新してgit pushします」と言う。でも会話が長くなると、最後にそれを実行しないことが増えた。次のセッションで「前回何をやっていたか」がわからなくなる。

**問題2: セッションクラッシュからの復旧**

Claude Codeがタイムアウトやエラーで終了した場合、次のセッションはゼロからのスタートになる。「どこまで進んでいたか」を毎回聞き返す必要があった。深夜の自律稼働中にクラッシュすると、朝まで気づかない。

この2つをHookで解決した。

**失敗した設計案1**: HANDOFF更新をClaudeのプロンプト指示だけで管理
→ プロンプトは忘れる。LLMへの口頭指示は信頼できない。

**失敗した設計案2**: 毎回の会話末尾にシステムプロンプトで「必ずgit pushせよ」と書く
→ 長い会話でコンテキストに埋もれて無視される。

**採用した設計**: `Stop` フックでセッション終了を確実に検知 → Node.jsスクリプトでgit pushする。Claude（LLM）に頼らない。インフラで保証する。

## 実装のポイント

### 1. session_end.js — セッション終了時の自動git push

実際のコード（`/root/ai-sales-company/.claude/hooks/session_end.js`）：

```javascript
const diff = execSync(
  'git diff --name-only HEAD',
  { cwd: REPO, timeout: 10000 }
).toString().trim();

const hasHandoff = diff.split('\n')
  .some(f => f.includes('HANDOFF') || f.includes('output/'));

if (hasHandoff) {
  execSync('git add -A', { cwd: REPO, timeout: 10000 });
  try {
    execSync('git diff --cached --quiet', { cwd: REPO, timeout: 5000 });
    // 差分なし → commit不要
  } catch (_) {
    // 差分あり → commit & push
    execSync(
      'git commit -q -m "[auto-session-end] HANDOFF/output更新"',
      { cwd: REPO, timeout: 15000 }
    );
    execSync('git push origin HEAD --quiet', { cwd: REPO, timeout: 20000 });
  }
}
```

ポイントは「差分チェックしてから commit する」点だ。`git diff --cached --quiet` は差分がなければ exit 0、あれば exit 1 を返す。この性質を使って空コミットを防いでいる。

**重要**: 末尾の `process.exit(0)` を省略すると Claude Code のUIがフリーズする。必ず入れること。

### 2. session_recovery.js — クラッシュ検出・自動復元

`crash_state.json` に `{ "cleanExit": false }` を常時書き込んでおき、正常終了時のみ `cleanExit: true` に更新する。次のセッション開始時（SessionStart Hook）に `cleanExit` が `false` のままなら「前回クラッシュした」と判定する。

```javascript
// crash_state.json（クラッシュ中の状態）
{
  "cleanExit": false,
  "lastActivity": "2026-06-30T07:05:28.000Z"
}
```

クラッシュ検出時は過去30件のツール操作ログと `session_checkpoint.md` の末尾20行を自動的にシステムプロンプトに注入する。「続きお願い」と言うだけで前回の作業が再開できる。

```javascript
// 復元コンテキストを標準出力に書き出す（Hookの仕組みでClaudeが読む）
let ctx = '【⚠️ 前回セッションは異常終了した可能性があります】\n';
ctx += `最終活動時刻: ${lastTs}\n\n`;
ctx += '## クラッシュ直前のツール操作ログ（末尾30件）\n```\n'
     + activityTail + '\n```\n\n';
ctx += '## session_checkpoint.md 末尾（ユーザー指示の履歴）\n```\n'
     + checkpointTail + '\n```\n';
process.stdout.write(ctx);
```

48時間以上前のクラッシュ痕跡は無視する（古すぎる情報を注入しない）。

### 3. detect_corrections.js — 教訓の自動記録

「違う」「ダメ」「やり直し」などの修正キーワードをリアルタイム検知する：

```javascript
const correctionWords = [
  '違う','ちがう','ダメ','だめ','やり直し','やりなおし',
  '間違い','間違ってる','そうじゃない','NG','ng'
];

const approvalWords = [
  '完璧','いい感じ','最高','これで行こう','ばっちり',
  'OKです','ありがとう','ぴったり'
];
```

検知したら `process.stdout.write` でClaudeへの指示を出力する。Claudeがその指示を読んで「教訓を記録しますか？」とオーナーに確認する。`/remember` コマンドで `lessons_learned.md` に永続保存される。

### 4. checkpoint.js — セッション継続性の担保

全ユーザー発言を `session_checkpoint.md` に自動追記する。30分以上の間隔があれば「新セッション」として扱いヘッダーを挿入する：

```javascript
const SESSION_GAP_MIN = 30;
const CHECKPOINT_INTERVAL = 5;

// 5件ごとにAI要約の追記を要求
if (counter.count % CHECKPOINT_INTERVAL === 0) {
  process.stdout.write('## 📌 AI要約 ' + ts + ' (#' + counter.count + ')\n');
  process.stdout.write('現在: [今やっていること1行]\n');
  process.stdout.write('次: [PCが落ちたとき最初にすべきこと1行]\n');
}
```

複数の会話セッションが1ファイルに時系列で記録される。スケジューラで深夜に起動されたエージェントの活動も全部ここに残る。

## この設計で解決されたこと

| 問題 | 解決策 | Hookの種類 |
|------|--------|-----------|
| HANDOFF更新忘れ | session_end.jsで自動push | Stop |
| クラッシュ後のコンテキスト消失 | session_recovery.jsで自動注入 | SessionStart |
| AIへの口頭指示が消える | checkpoint.jsで全会話を永続化 | UserPromptSubmit |
| 同じミスを繰り返す | detect_corrections.jsで教訓検知 | UserPromptSubmit |

LLMへの「指示」ではなくインフラへの「設計」で解決した。これがHookシステムの本質だ。

## 次の章へ

第4章では、このシステムの「心臓部」であるスケジューラ（`agent_dispatcher.js`）を解説する。毎晩深夜に自律稼働している仕組み — 何時に何のエージェントを何の条件で起動するか — の実装と設計判断を全公開する。
