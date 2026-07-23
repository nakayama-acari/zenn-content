---
title: "Claude Code Hooksの実装ガイド — セッション終了時に自動でgit pushする仕組みをコードで解説"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "ai", "automation", "hooks", "nodejs"]
published: true
---

## `.claude/hooks/` に9本のスクリプトがある

このAI営業会社には常駐プロセスが3つしかない（`ai-office`・`vsync-crm`・`accea-crm`、PM2管理）。だが実際に会社の「習慣」を作っているのは、常駐すらしていない9本のNode.js/PowerShellスクリプトだ。

```bash
$ ls .claude/hooks/
check_handoff.ps1      detect_corrections.ps1  session_counter.json
checkpoint.js          log_file_saves.js       session_end.js
crash_state.json       log_tool_activity.js    session_recovery.js
detect_corrections.js  save_tool_memory.js     tool_activity.log
```

このうち`.js`が7本、`.ps1`が2本。呼ばれたときだけ起動し、数百ミリ秒で終了する。この記事では、実際に動いている2本のコードを引きながら「Claude Codeの自律稼働を支えるHookの実装」を解説する。

---

## Hookは「Claude Codeへの命令」ではなく「OSイベントへの割り込み」

Claude Codeの`settings.json`には`hooks`セクションがあり、`SessionStart`（セッション開始）・`UserPromptSubmit`（発言送信時）・`PreToolUse`（ツール実行前）・`Stop`（セッション終了）といったイベントに任意のコマンドを紐付けられる。

```json
{
  "hooks": {
    "UserPromptSubmit": [
      { "hooks": [
        { "type": "command", "command": "node .claude/hooks/checkpoint.js" },
        { "type": "command", "command": "node .claude/hooks/detect_corrections.js" }
      ]}
    ],
    "Stop": [
      { "hooks": [
        { "type": "command", "command": "node .claude/hooks/session_end.js" }
      ]}
    ]
  }
}
```

ポイントは、Claudeという**LLMにやらせるのではなく**、イベント発生時にOSレベルで確実にスクリプトを起動させる点だ。プロンプトで「必ずHANDOFF.mdをgit pushしてください」と何度書いても、長い会話ではその指示がコンテキストの奥に埋もれて実行されないことがある。Hookはそれを起こさせない。

---

## 実装1: `session_end.js` — セッション終了を検知して自動push

実際に動いているコード（`/root/ai-sales-company/.claude/hooks/session_end.js`）はこうなっている。

```javascript
const diff = execSync('git diff --name-only HEAD', { cwd: REPO, timeout: 10000 }).toString().trim();
const hasHandoff = diff.split('\n').some(f => f.includes('HANDOFF') || f.includes('output/'));

if (hasHandoff) {
  execSync('git add -A', { cwd: REPO, timeout: 10000 });
  try {
    execSync('git diff --cached --quiet', { cwd: REPO, timeout: 5000 });
    // 差分なし → commit不要
  } catch (_) {
    execSync('git commit -q -m "[auto-session-end] HANDOFF/output更新"', { cwd: REPO, timeout: 15000 });
    execSync('git push origin HEAD --quiet', { cwd: REPO, timeout: 20000 });
  }
}
process.exit(0);
```

`git diff --cached --quiet` はステージ済み差分がなければ exit 0、あれば exit 1 を返す。この終了コードの違いを`try/catch`で拾って「差分があるときだけcommitする」判定に使っている。空コミットでgitログを汚さないための最小限の分岐だ。

最後の`process.exit(0)`は省略できない。実際、社内のCLAUDE.mdには「フックのJSを編集するとき：末尾の`process.exit(0)`を省略しない（省略するとUIがフリーズする）」と赤字で書いてある。フックはClaude Codeのメインプロセスと標準入出力でやり取りしており、明示的にプロセスを終了させないとClaude Code側が応答待ちのままフリーズする。一度これで開発中のセッションが固まり、ターミナルごと再起動する羽目になった。

---

## 実装2: `checkpoint.js` — 会話をまたいだ記憶を作る

セッションが終われば会話のコンテキストは消える。次回起動時に「前回何をしていたか」を知る手段が必要だ。`checkpoint.js`は発言のたびに`session_checkpoint.md`へ追記し、一定間隔でAIに要約を書かせる。

```javascript
const SESSION_GAP_MIN = 30;
const CHECKPOINT_INTERVAL = 5;

// 前回更新から30分以上経っていれば「新セッション」とみなす
const gapMin = (now - lastUpdateDate) / 60000;
if (gapMin <= SESSION_GAP_MIN) {
  counter = Object.assign({}, counter, saved);
  isNewSession = false;
}

// 5件ごとにAI要約の追記を要求
if (counter.count % CHECKPOINT_INTERVAL === 0) {
  process.stdout.write('【🔖 定期チェックポイント】\n');
  process.stdout.write('現在: [今やっていること1行]\n');
  process.stdout.write('次: [PCが落ちたとき最初にすべきこと1行]\n');
}
```

`process.stdout.write`で出力した文字列は、Claude Codeがそのままシステムプロンプトの一部として読み込む。つまりHookは「スクリプトを実行する」だけでなく「Claudeに次に何をさせるかを指示する」チャンネルでもある。5件ごとに要約を書かせているのは、コンテキストが長くなりすぎる前に定期的に現在地を記録させるためだ。実際、この記事を書いている最中にもこの仕組みが発火して、AI要約の追記を求められた。

30分の間隔判定にも理由がある。深夜のスケジューラ起動は前回の対話セッションと無関係なので「新セッション」として区切りたい。逆に数分おきの短いやり取りは同じセッションとして連結したい。`lastUpdate`のタイムスタンプ差分だけで、この2つを区別している。

---

## 実装3: `detect_corrections.js` — 「違う」を聞き逃さない

オーナーが「違う」「ダメ」「やり直し」と言ったとき、それは単なる会話ではなく訂正のシグナルだ。これを取りこぼすと同じミスを繰り返す。

```javascript
const correctionWords = [
  '違う','ちがう','ダメ','だめ','やり直し','間違い','そうじゃない','NG'
];
const approvalWords = [
  '完璧','いい感じ','最高','ばっちり','ありがとう','ぴったり'
];

if (foundCorrection) {
  process.stdout.write('【🔴 修正キーワード「' + foundCorrection + '」検出】\n');
  process.stdout.write('「これを教訓として記録しますか？」とオーナーに確認してください。\n');
}
```

キーワードを検知したらHookが自らgit commitするわけではない。あくまで「Claudeに確認を促す」だけに留めている。記録するかどうかの最終判断はオーナーに委ねる設計だ。承認判定した場合は`/remember`コマンドで`lessons_learned.md`に永続化される。この会社の全19エージェントが持つ「教訓ファイル」は、ほぼすべてこの仕組み経由で溜まっている。

---

## なぜ全部Hookに寄せたのか

最初はHookなしで、プロンプトの指示文だけで運用していた。壊れ方は2パターンだった。

1. 会話が長くなるとHANDOFF更新の指示が読み飛ばされる
2. セッションがクラッシュすると、次回起動時に前回の状況が完全にわからなくなる

どちらも「LLMに毎回思い出させる」方式では解決しなかった。今の設計は逆で、**Claudeの記憶力に依存する部分を極力減らし、OSイベントで確実に発火するインフラ側に処理を移した**。クラッシュ検知も同様で、`crash_state.json`に`cleanExit: false`を常時書き込み、`session_end.js`が正常終了時だけ`true`に上書きする。次回`SessionStart`で`cleanExit`が`false`のままなら「前回クラッシュした」と機械的に判定できる。実際、このHANDOFFやこの記事を書いている今のセッションも、直前セッションのクラッシュを`session_recovery.js`が検知して復旧ログを自動で読み込ませた状態から始まっている。

---

## この会社のHook・スケジューラ・全19エージェント設計を解説する本

`.claude/hooks/`全ファイルの実装・クラッシュ復旧の完全なロジック・`.claude/agents/`19体の設計パターンを、実際のコードと設定ファイルつきで解説するZennの有料本を書いている。

**[Claude Code で作る AI 自動化会社 完全解説](https://zenn.dev/nakayama_acari/books/ai-company-automation)**（¥4,980）

架空のサンプルコードではなく、今この瞬間動いているVPS上のファイルをそのまま載せている。

