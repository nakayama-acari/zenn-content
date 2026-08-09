---
title: "第9章: Hookシステムの続き — クラッシュ復旧・活動ログ・セッションメモリ、まだ見せていなかった3本"
---

## この章で学ぶこと

- クラッシュしたセッションを、次回起動時にどう「復旧」させているか（`session_recovery.js`）
- ツール実行のたびに何が起きているか（`log_tool_activity.js` の二重役割）
- `/compact` でコンテキストが消えても情報を残す仕組み（`save_tool_memory.js`）
- これら3本は、以前の集客記事「Hooksの実装ガイド」では触れていない部分だ

`.claude/hooks/` には9本のスクリプトがある。前回は `session_end.js`・`checkpoint.js`・`detect_corrections.js` の3本を解説した。今回はその続きで、まだコードを見せていなかった残り3本を扱う。直近1ヶ月でスキ数が最も伸びたのがHooks系の記事だったため、同じテーマをもう一段掘り下げる。

## 実際のシステム構成

3本はすべて `PostToolUse`（ツール実行後）または `SessionStart`（セッション開始時）で発火する。役割はきれいに分かれている。

```
PostToolUse（全ツール）  → log_tool_activity.js  … 生存マーカー更新 + AI Office反映
PostToolUse（Read/Bash等）→ save_tool_memory.js   … ツール結果をファイルへ退避
SessionStart             → session_recovery.js    … クラッシュ検知 + 復旧コンテキスト注入
```

`log_tool_activity.js` は毎ツール呼び出し後に必ず動く軽量ログで、`tool_activity.log` に1行追記しながら `crash_state.json` の `lastActivity` を更新する。

```javascript
try {
  fs.writeFileSync(STATE_FILE, JSON.stringify({
    lastActivity: now.toISOString(),
    cleanExit: false,
    sessionId: obj.session_id || ''
  }, null, 2), 'utf8');
} catch (e) {}
```

ポイントは `cleanExit: false` を**毎回**書き込んでいることだ。正常終了時にこれを `true` に上書きするのは `session_end.js` の役目（前回記事で紹介済み）。つまり `cleanExit` が `false` のまま次のセッションが立ち上がったら、間に正常終了のタイミングが一度もなかった＝クラッシュした、と機械的に判定できる。

## なぜこの設計にしたか

**クラッシュ検知だけでは足りない。「何が起きていたか」まで復元しないと使えない。**

`session_recovery.js` は `SessionStart` で `crash_state.json` を読み、`cleanExit !== false` なら何もせず終了する。クラッシュ痕跡があった場合だけ、次の2つを合成して注入する。

```javascript
const activityTail = tail(LOG_FILE, 30);        // tool_activity.log 末尾30件
const checkpointTail = tail(CHECKPOINT_FILE, 20); // session_checkpoint.md 末尾20件
```

`tool_activity.log` は「直前まで何のツールをどう使っていたか」の機械ログ、`session_checkpoint.md` は「ユーザーが何を指示していたか」の会話ログ（第8章で触れた `checkpoint.js` が書いている）。前者だけだと「Bashで何かを実行していた」までは分かっても目的が分からない。後者だけだと「〇〇をやってと言われた」までは分かってもどこまで進んだか分からない。2つを合わせて初めて「前回、何をどこまでやっていたか」が復元できる。

```javascript
function emit(ctx) {
  process.stdout.write(JSON.stringify({
    hookSpecificOutput: { hookEventName: 'SessionStart', additionalContext: ctx }
  }));
}
```

出力形式が `checkpoint.js`（プレーンテキストの `stdout.write`）と違い、JSON構造化された `hookSpecificOutput.additionalContext` になっている点に注意したい。`SessionStart` イベントはこの形式でないとClaude Code側にコンテキストとして正しく取り込まれない。イベントの種類によって出力プロトコルが違うのは、このシステムのHookを書くときに毎回引っかかるポイントだ。

**48時間という閾値は、事故から逆算して決めた値ではなく先回りして決めた値。**

```javascript
const STALE_HOURS = 48; // これより古いクラッシュ痕跡は無視
```

もし1週間前のクラッシュ痕跡がいつまでも残り続けて、無関係な新しいセッションに「前回クラッシュしました」と毎回注入されたら、逆にノイズになる。48時間より古い `lastActivity` は無視してそのまま素通りする設計にしてある。

**二重注入を防ぐために、注入した瞬間にフラグを倒す。**

```javascript
state.cleanExit = true;
state.recoveredAt = new Date().toISOString();
fs.writeFileSync(STATE_FILE, JSON.stringify(state, null, 2), 'utf8');
```

復旧コンテキストを注入したら、その場で `cleanExit` を `true` に書き換える。これをやらないと、同じセッション内で複数回 `SessionStart` 相当の処理が走ったとき（`/clear` 等）に同じ復旧メッセージが繰り返し出てしまう。

## 実装のポイント

### ①「読み取り専用ログ」のつもりが「本番機能への書き込み経路」になっていた

`log_tool_activity.js` には、ログを書くだけでは終わらないもう一つの役割がある。2026-07-23に追加された、AI Office（監視ダッシュボード）へのHTTP通知だ。

```javascript
const hardTimer = setTimeout(finish, 600); // office-appが無応答でも600ms以上は絶対に待たない
```

もともとAI Office上の「吹き出しクリックで作業ログ閲覧」機能は、AI Office経由で動くセッション（チャット・ディベート・案件・朝会）だけを対象にしていた。だが実際の日常業務の大半は、このHookが動いている外部ターミナルセッション（今この章を書いているセッションもそう）で行われている。そこが対象外だったとオーナー指摘で気づき、`log_tool_activity.js` 自身に通知役を持たせた。

全ツール呼び出しのたびに動くログが、外部サービス（office-app、port 3000）への同期HTTPリクエストでブロックしたら本末転倒になる。だから `600ms` で強制的に打ち切る `hardTimer` を必ずセットし、`finish()` は多重呼び出しされても1回しか `process.exit(0)` しないよう `settled` フラグでガードしてある。office-appが落ちていても、このセッションの体感速度には一切影響しない設計だ。

### ② セッションメモリへの書き込みには「注入対策」が要る

`save_tool_memory.js` は `Read`・`Bash`・`mcp__fetch__fetch` の結果を `output/session_memory/YYYYMMDD.md` に自動で書き出す。`/compact` でコンテキストが圧縮されても、このファイルを読み直せば失われた情報を復元できる。

ここで面白いのは `sanitize()` 関数だ。

```javascript
function sanitize(str) {
  return String(str)
    .slice(0, 200)
    .replace(/[\x00-\x1F\x7F`#]/g, ' ')  // 危険文字を空白に置換
    .trim()
    .slice(0, 120);
}
```

ツールの実行結果（Webページの中身やコマンド出力）を無加工でMarkdownファイルに埋め込むと、そこに偶然含まれる改行やバッククォート・`#` が、書き出し先のMarkdown構造そのものを壊しうる。コメントにある通り「万一改行が入った場合の見出し注入を防ぐ」ための対策で、外部から取得したテキストをそのままファイルに書き込む処理には毎回このクラスの検討が要る。日本語や絵文字はホワイトリスト方式にすると壊れるため、あくまで危険な制御文字だけを狙い撃ちしている。

保存対象も絞ってある。

```javascript
const SKIP_PATHS = [
  'session_memory', 'HANDOFF.md', 'session_checkpoint.md',
  'crash_state.json', 'tool_activity.log',
];
```

自分自身のファイルを読み返しても保存しない（循環保存の防止）。HANDOFF等はそもそも永続ファイルなので二重保存する意味がない。

### ③ 3本とも共通しているルール：ノイズを出さない設計

`save_tool_memory.js` は短いBashコマンド（15文字未満）やレスポンスが30文字未満のものをスキップする。`log_tool_activity.js` は `agent_type` が無ければAI Office通知処理自体に入らない。`session_recovery.js` は48時間ルールで古い痕跡を無視する。

3本に共通しているのは「常時発火するHookだからこそ、意味のない情報で埋め尽くさない」という判断基準だ。毎ツール呼び出しごとに発火する仕組みは便利な反面、無条件に全部記録すると肝心な情報が埋もれる。「保存する条件」を明示的に絞ることも、Hook設計の一部になっている。

## この章のまとめ

| Hook | 発火タイミング | 役割 |
|---|---|---|
| `log_tool_activity.js` | PostToolUse（全ツール） | 生存マーカー更新 + AI Officeへの活動反映（600ms打ち切り） |
| `save_tool_memory.js` | PostToolUse（Read/Bash/fetch） | ツール結果をファイルへ退避（`/compact` 後も参照可能） |
| `session_recovery.js` | SessionStart | クラッシュ検知 + 活動ログ・チェックポイントの合成注入 |

前回の記事で紹介した3本（`session_end.js`・`checkpoint.js`・`detect_corrections.js`）と合わせて、これで `.claude/hooks/` の主要6本のロジックを一通り解説したことになる。9本のうち残りは `.ps1`（Windows環境向け）と、ログ・状態を保持するだけの非スクリプトファイルだ。

## 次の章へ

次章以降も、直近でスキ・PVが伸びたテーマを深掘りする形で更新していく。
