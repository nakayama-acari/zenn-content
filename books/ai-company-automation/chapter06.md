---
title: "第6章: MCPサーバー30余りの実態 — 繋がっているのに使えない瞬間をどう乗り切るか"
---

## この章で学ぶこと

- このシステムが実際に接続している30以上のMCPサーバー（GitHub・Playwright・Slack・検索系・Google系）の使い分け
- MCPサーバーは「接続済み」と「ツールが呼べる」が同じ意味ではないという実際の挙動
- 書き込み系だけが認証エラーで落ちる、読み取りだけ成功するという非対称な故障の実例
- 通知が失敗しても気づけない「無音失敗」をどう潰すか

## 実際のシステム構成

### 30以上のMCPサーバーは「起動＝即使える」ではない

このシリーズで何度も「GitHub MCP」「Playwright MCP」と書いてきたが、実際にセッションが立ち上がる瞬間の内部はこうなっている。

```
The following MCP servers are still connecting — their tools
(typically named mcp__<server>__*) are not yet available but
will appear shortly:
brave-search / claude.ai Notion / claude.ai Slack / duckdb /
exa / fetch / firecrawl / github / github-trending /
google-sheets / hourei / playwright / tavily
```

これは架空の例ではなく、この章を書いている今のセッション起動時に実際に表示された内容そのままだ。13個のサーバーが「まだ接続中」の状態で並び、GitHubやPlaywrightのように毎日使っているサーバーすら、起動直後は使えない。

さらに、接続済みのサーバーでも、ツールの詳細（どんな引数を渡せばいいか）は最初から全部読み込まれているわけではない。

```
The following deferred tools are now available via ToolSearch.
Their schemas are NOT loaded — calling them directly will fail
with InputValidationError. Use ToolSearch with query
"select:<name>[,<name>...]" to load tool schemas before calling them:
CronCreate / mcp__claude_ai_Gmail__create_draft /
mcp__claude_ai_Google_Calendar__create_event / ...
```

つまり、道具箱の引き出しを開けても、中に入っているのは道具の「名前が書かれた札」だけの状態がある。実際に使うには、その名前を指定して「詳しい使い方（スキーマ）を持ってきて」と一度呼び出す必要がある。名前だけ知っていても、そのままでは使えない。これがこのシステムの実際の起動シーケンスであり、30以上という数のツールを常時全部フル読み込みしておくと重くなりすぎるための設計だと理解している。

### 実際に使っているMCPの一覧（CLAUDE.md抜粋）

```markdown
## 利用可能な共有スキル（内容に応じ自動起動 or 名前指定）
- Web/情報: fetch / firecrawl / tavily / exa / brave / feedparser
- データ: duckdb(MCP+CLI) / xlsxスキル / csvkit・visidata
- 法令: hourei(MCP・e-Gov)
- クラウド: Google Drive MCP
```

検索系だけで5系統（fetch・firecrawl・tavily・exa・brave）が並んでいるのは重複ではない。Brave Searchが特定のパラメータで422エラーを返すことが分かっていて、その場合はリトライせずtavily・exaに即切り替える、という運用ルールが実際にfukugyo_publisher.mdに明記されている。1つのツールが壊れても業務を止めないための冗長構成だ。

## なぜこの設計にしたか

**失敗した設計案1：MCPが繋がらなければ即座にエラーで止める**
→ 実際には「接続中でまだ使えない」状態と「本当に壊れている」状態を区別する必要がある。前者は数秒待てば直るのに、それを故障扱いにすると誤報が増える。

**失敗した設計案2：全ツールのスキーマを起動時に全部読み込む**
→ ツール数が多いシステムほど起動が重くなる。必要になった時点で名前から詳細を取りに行く「遅延読み込み」方式にすることで、起動直後から動き始められる。

**失敗した設計案3：読み取りと書き込みは同じ経路で同じ結果になるはず**
→ これは次で説明する通り、実際には崩れることがあった。

## 実装のポイント

### 読み取りは通るのに、書き込みだけ認証エラーになる非対称な故障

Zenn本の第5章をpushしようとしたとき、GitHub用のMCPで実際に起きたのはこれだった。

```
mcp__github__get_file_contents  → 成功（章の一覧が取得できる）
mcp__github__push_files         → Authentication Failed: Requires authentication
mcp__github__create_or_update_file → Authentication Failed: Requires authentication
```

同じ認証情報を使っているはずなのに、読み取り系のツールは通るのに書き込み系だけが軒並み失敗する。これは「MCPサーバーが落ちている」わけではなく、「トークンに書き込み権限（スコープ）が足りていない」ときに起きる典型的な症状だと分かった。

対処は、原因の切り分けを待たずに手を止めないことだった。

```bash
# ローカルにcloneしたリポジトリに直接pushする（MCPの書き込みが直るまでの回避策）
cd /root/zenn-content
git add books/ai-company-automation/chapter06.md books/ai-company-automation/config.yaml
git commit -m "feat(zenn): 第6章 MCP活用術 を追加"
git push origin main
git ls-remote origin main   # プッシュ後、リモートのHEADが一致しているか必ず確認
```

MCPの書き込みが直ることを待つのではなく、同じ内容を別の経路（ローカルclone＋直接push）で確実に届ける。届いたかどうかは`git ls-remote`でリモート側のコミットハッシュを見て確認する。「push成功」というログの文字列を信じるのではなく、リモートの実物を確認する、という一手間がここでも効いている。

### 通知だけが無言で失敗する「missing_scope」

X投稿・note投稿が成功した後、Slackに完了通知を送る処理も同じ根っこの問題を抱えていた。

```javascript
// notifySlack() の中身（要旨）
const dm = await post({ path: '/api/conversations.open', ... }, JSON.stringify({ users: SLACK_USER_ID }));
if (!dm.ok || !dm.channel?.id) return; // ← ここで無言でreturnしていた
```

`conversations.open`（DMを開く処理）が`missing_scope`（Bot Tokenに必要な権限が付いていない）で失敗すると、この関数はエラーも投げずに黙って終了する。投稿自体は成功しているのに、Slackには何も届かない。ログを見なければ「通知が来ない＝何も起きていない」と誤解してしまう。

再発防止としてやっているのは単純だ。投稿系の処理を実行したら、Slackに通知が届いたかを毎回目で確認する。届いていなければ、スクリプト内部の通知に頼らず、別経路のSlack送信で必ず代替する。「自動で通知されるはず」を過信しないことが、無音失敗を防ぐ一番確実な方法だった。

## この章のまとめ

| 課題 | 対応状況 |
|------|------|
| MCPサーバーが「接続中」で一時的に使えない | 仕様として理解し、エラーと誤診断しない |
| ツールのスキーマが遅延読み込みされる | 名前だけの状態から必要時に詳細を取りに行く運用に対応済み |
| 書き込みだけ認証エラーになる非対称故障 | ローカルclone経由の直接push＋リモート確認で回避継続中（根本解決はトークンのスコープ見直し待ち） |
| 通知の無音失敗（missing_scope） | 毎回目視確認＋手動送信で代替。恒久対応は未着手 |

30以上のツールを繋いでいるという数字だけを見ると立派なシステムに見える。だが実態は、その半分近くが起動直後は「名前だけ」の状態で、書き込み系だけが原因不明で落ち続け、通知は無言で消える。それでも回せているのは、ツールを過信せず、届いたかどうかを都度確認する手順を諦めずに続けているからだ。

## 次の章へ

次章では、このシステムの収益化パイプライン（BOOTH・note有料記事・Gumroad・SEO記事の実際の売上と、3週連続0件だった有料記事にどう向き合ったか）を数字ごと公開する。
