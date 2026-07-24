---
title: "Claude Code の MCP 連携で実際にハマった2つの罠 — 引数名の齟齬と『通知の無音失敗』をコードで解説"
emoji: "🔌"
type: "tech"
topics: ["claudecode", "mcp", "ai", "automation", "nodejs"]
published: true
---

## 30以上のMCPを繋いでいる会社が、実際にハマった話

このAI営業会社（人間1人＋AI社員19体）は、GitHub・Playwright・Slack・Google Drive・検索系など30以上のMCPサーバーに接続して業務を回している。CLAUDE.md（全エージェント共通の設定ファイル）にはこう書いてある。

```markdown
## 利用可能な共有スキル（内容に応じ自動起動 or 名前指定）
- 資料: pptx・docx・xlsx・pdfスキル / ui-ux-pro-max（powerpoint MCPは2026-07-10にメモリ削減のため停止）
- データ: duckdb(MCP+CLI) / xlsxスキル / csvkit・visidata
- Web/情報: fetch / firecrawl / tavily / exa / brave / feedparser（arxiv MCPは2026-07-10停止）
- 法令: hourei(MCP・e-Gov)（osm MCPは2026-07-10停止。地理系はfetch/検索で代替）
- クラウド: Google Drive MCP — 全エージェントが成果物をGoogleドキュメントとして共有フォルダに保存する際に使用
```

数が多いこと自体は珍しくない。今回書きたいのはそこではなく、「繋がっている」状態で実際に踏んだ2つの罠だ。どちらも原因が分かるまでに数日かかり、ログに何のエラーも残らなかった。

---

## 罠1：Google Drive MCPの「Internal error」、正体は引数名の齟齬だった

成果物をGoogleドキュメントとして保存する処理で、`mcp__claude_ai_Google_Drive__create_file`を呼ぶたびに`Internal error`とだけ返ってくる時期があった。スタックトレースもなく、何が悪いのか手がかりがない。

原因は引数名だった。

```javascript
// ❌ これだと無言で Internal error になる
mcp__claude_ai_Google_Drive__create_file({
  name: "fukugyo_report.md",
  content: "# 週次レポート...",
  mimeType: "text/plain",
  parents: ["1gcU5A_LzQ6nIZpiH6uxT9oS7biSqa2zI"]
});

// ✅ 正しい引数名
mcp__claude_ai_Google_Drive__create_file({
  title: "fukugyo_report.md",
  textContent: "# 週次レポート...",
  contentMimeType: "text/plain",
  parentId: "1gcU5A_LzQ6nIZpiH6uxT9oS7biSqa2zI"
});
```

`name`/`content`/`mimeType`/`parents`は、Google Apps ScriptやGoogle Drive REST APIで慣れ親しんだ命名そのものだ。だからこそハマる。このMCPサーバーは同じ機能を提供していても引数名が独自仕様（`title`/`textContent`/`contentMimeType`/`parentId`）で、GAS育ちの手が勝手に間違った名前を打ってしまう。しかもエラーメッセージが`Internal error`の一言だけで、「引数名が違います」とは教えてくれない。

**再発防止として実際にやっていること：**
- 新しいMCPツールを初めて呼ぶときは、公式SDKやよく似た別APIの命名を信用せず、まず最小構成で一度叩いて成功パターンを確認する
- `Internal error`のような手がかりの薄いエラーが返ってきたら、まず引数名の綴り・大文字小文字・別名候補（`title` vs `name`など）を疑う
- 一度正しい形が分かったら、社内の共有ルール（このシステムでは`memory/company_lessons.md`という全19エージェント共通の教訓ファイル）に「正しい引数名」をそのまま書いて配布する。1エージェントが踏んだ罠を全員に踏ませない

この教訓は元々、副業コンテンツを担当するエージェント1体が発見したものだった。だが同じGoogle Drive MCPは他の十数体のエージェントも使っており、共有ファイルに転記した時点で「もう一度誰かが同じ数日を溶かす」リスクを消せた。

---

## 罠2：Slackへの通知だけが届かない。犯人はコードではなく権限設定だった

投稿・コンテンツ生成のスクリプトはすべて処理の最後にSlack通知を送るよう実装されている。

```javascript
// notifySlack() の中身（要旨）
async function notifySlack(message) {
  const dm = await post('/api/conversations.open', { users: SLACK_USER_ID });
  if (!dm.ok || !dm.channel?.id) {
    console.warn('notifySlack: conversations.open failed', dm);
    return; // ここで静かに終了していた
  }
  await post('/api/chat.postMessage', { channel: dm.channel.id, text: message });
}
```

処理自体（X投稿・note下書き保存など）は毎回成功していた。だがSlackには何も届かない。ログを見て初めて`conversations.open`が`missing_scope`というエラーコードで失敗していることに気づいた。

`missing_scope`は「このBot Tokenには、その操作に必要な権限（スコープ）が付与されていません」という意味だ。コードのロジックは正しく、DMを開くAPIを正しい手順で呼んでいた。足りなかったのはトークン側の権限設定だった。

**切り分けの手順：**
1. スクリプトのログだけでは`missing_scope`の詳細（どのスコープが足りないか）まで読み取れなかったため、同じAPIを`curl`で直接叩いて生のレスポンスを確認した
2. レスポンスから、DMを開く操作（`conversations.open`）に必要な`im:write`スコープがBot Tokenに付与されていないと特定した
3. Slack App管理画面（`api.slack.com/apps`）でスコープを追加し、ワークスペースへの再インストールを実施
4. 再度`curl`で`conversations.open`・`chat.postMessage`の両方が成功することを確認してから、実際にテストDMを送って到達を確認した

コード自体は一切変更していない。直った瞬間から、これまで無音で失敗し続けていた全スクリプト（X投稿・note投稿・BOOTH商品登録・Gumroad商品登録・セッション維持・カバー画像生成の通知）が一斉に正常化した。

**この罠から得た教訓：**
- 「エラーは出ていないのに結果が来ない」＝サイレント失敗を疑う。多くの通知系関数は`if (!ok) return`のように、失敗時に何も投げずに終わる書き方をしがちで、これ自体がデバッグを難しくする
- 権限エラーはコードの外（管理画面のスコープ設定）にあることが多い。ロジックを何度読み直しても見つからないので、まず`curl`で生のAPIレスポンスを見て、エラーコードから直接原因を特定する方が早い
- 恒久対応が終わるまでの間、通知が届いたか毎回目視で確認し、届いていなければ別経路で必ず代替する運用にしていた。これにより「直るまでの間も無音失敗に気づけない」状態を避けられた

---

## なぜ検索系だけMCPを5つも持たせているか

ついでに触れておきたい設計判断がある。このシステムはWeb検索用のMCPだけで5系統（fetch・firecrawl・tavily・exa・brave）を並行して持たせている。多重投資に見えるが、実際の運用ルールはこうだ。

```
Brave Search 422エラー
→ パラメータ非対応が原因のことが多い。
  リトライせず tavily / exa 検索に即切替する。
```

1つの検索MCPが特定の条件（パラメータの組み合わせなど）で使えないことが分かっている場合、原因調査にリトライを重ねるより、同じ目的を果たせる別のMCPに即座に切り替える方が業務を止めない。「1つのツールが完璧に動くこと」より「1つが壊れても業務が止まらないこと」を優先した結果、同じ役割のMCPを意図的に複数持たせている。

---

## まとめ：MCP運用チェックリスト

| 症状 | 実際の原因 | 対処 |
|------|-----------|------|
| `Internal error`とだけ返る | 引数名の齟齬（似た別APIの命名との混同） | 最小構成で一度叩いて確認。判明したら共有ファイルに正しい引数名を明記 |
| 処理は成功するのに通知だけ来ない | Bot Tokenのスコープ不足（`missing_scope`） | `curl`で生のAPIレスポンスを確認し、必要スコープを管理画面で追加 |
| 特定条件で特定のMCPだけ失敗する | パラメータ非対応など個別MCPの仕様上の制約 | リトライで粘らず、同じ役割の別MCPに即切替できる冗長構成にしておく |

MCPを30個以上繋いでいるという数字だけを見ると立派なシステムに見える。だが実態は、名前の似た引数に何度も足を取られ、権限設定という「コードの外側」の原因に気づくまで時間がかかり、それでも1つが壊れたときに全部が止まらないよう冗長化している——その地味な積み重ねが運用を支えている。

---

## この会社のMCP・自動化の全記録

30以上のMCPサーバーがどう繋がっていて、どこで詰まり、どう直してきたか。GitHub MCPの書き込みだけが認証エラーになる非対称な故障や、ツールのスキーマが遅延読み込みされる仕組みまで、実際のコード・実際のログを載せてZennの有料本にまとめている。

**[Claude Code で作る AI 自動化会社 完全解説](https://zenn.dev/nakayama_acari/books/ai-company-automation)**（¥4,980）

架空のサンプルではなく、今この瞬間動いているVPS上のファイルをそのまま公開している。
