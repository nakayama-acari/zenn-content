---
title: "第10章: GitHub MCPが24日間認証エラーのまま記事を毎週届け続けた理由"
---

## この章で学ぶこと

- GitHub MCPと.envが「別々の設定空間」である構造を理解する
- フォールバック（ローカルclone経由のgit push）の実装パターン
- 「エラーが出ていても本番が止まらない」ための二重化設計と、その落とし穴

## 実際に起きたこと

2026年7月24日から、ZennへのMCP経由push（`mcp__github__create_or_update_file`）が毎回同じエラーで失敗するようになった。

```
Authentication Failed: Requires authentication
```

奇妙なのは、同じトークンを使って`curl`で直接GitHub APIを叩くと正常に通ることだ。MCPだけが通らない。

最初の診断では「MCPサーバープロセスが旧設定のまま起動していて、後から.envに追加したトークンを拾えていない」と判断した。それ自体は正しい推測ではあったが、より根本的な原因を見落としていた。

## 原因：MCPサーバーは .env を読まない

約17日後にsystem部の調査で判明した根本原因はシンプルだった。

`/root/ai-sales-company/.env`には正しいGitHub PATが設定されている：

```
GITHUB_TOKEN=ghp_xxxxx（実際のトークン）
```

しかし、GitHub MCPサーバーはこの.envファイルを一切参照しない。MCPサーバーに渡される環境変数は、`~/.claude.json`の`env`セクションで明示的に指定する必要がある。

```json
// 修正前（24日間気づかなかった状態）
{
  "mcpServers": {
    "github": {
      "command": "mcp-server-github",
      "args": [],
      "env": {
        "GITHUB_TOKEN": "YOUR_GITHUB_PAT_HERE"
      }
    }
  }
}
```

初期セットアップ時に`YOUR_GITHUB_PAT_HERE`というプレースホルダーが入ったまま更新されていなかった。

「`curl`は通るのにMCPは通らない」の理由はここにある。`curl`はシェル経由で`.env`を展開したスクリプトから呼んでいる。MCPサーバープロセスは`~/.claude.json`の`env`セクションから環境変数を受け取る。まったく別の場所を見ている。

## なぜ24日間気づかなかったのか

3つの要因が重なった。

**① 読み取りは正常だった**  
`mcp__github__get_file_contents`（ファイルの読み取り）は公開リポジトリへのアクセスなので認証なしでも200を返す。書き込み系だけが失敗する。「MCPが動いている」という誤認が生まれやすかった。

**② フォールバックが毎回成功していた**  
書き込み失敗を検知したら、ローカルclone（`/root/zenn-content`）経由でgit pushに切り替えるフローを取っていた。最終的なZennへの反映は毎回成功するため、「問題はあるが本番は動いている」状態が続いた。

**③ その状態を「解決済み」と誤認しかけた**  
フォールバックが安定して動いていると、根本原因の解消が後回しになる。実際、「ローカルcloneで回避できているからまあいい」という意識が働いた期間があった。

## フォールバック実装：ローカルclone経由

書き込み失敗を検知したときのフォールバック手順はシンプルだ：

```bash
# /root/zenn-content がローカルclone
cd /root/zenn-content

# 最新状態に同期（乖離がある場合）
git fetch origin main
git log origin/main..HEAD --oneline  # ローカルが先行していないか確認

# 新章ファイルとconfig.yamlを追加してコミット
git add books/ai-company-automation/chapter10.md
git add books/ai-company-automation/config.yaml
git commit -m "feat(zenn): 第10章 GitHub MCP認証エラーとフォールバック設計 を追加"
git push origin main
```

注意点がひとつある。複数のセッションがそれぞれローカルcloneを持っていて、それぞれが別々のコミットをpushしようとする場合、non-fast-forwardで弾かれることがある。そのときは一時worktreeを使う：

```bash
# /root/tmp_worktrees/ に一時worktreeを作成
git worktree add /root/tmp_worktrees/zenn_chapter10 origin/main

# worktree内で作業
cd /root/tmp_worktrees/zenn_chapter10
cp /path/to/chapter10.md books/ai-company-automation/
git add .
git commit -m "feat(zenn): 第10章を追加"
git push origin HEAD:main

# 後始末
cd /root/zenn-content
git worktree remove /root/tmp_worktrees/zenn_chapter10
```

## 修正：~/.claude.jsonに実トークンを設定する

修正の内容は単純だった。`~/.claude.json`のGITHUB_TOKENをプレースホルダーから実トークンに書き換える。

```python
import json

with open('/root/.claude.json', 'r') as f:
    config = json.load(f)

config['mcpServers']['github']['env']['GITHUB_TOKEN'] = 'ghp_your_real_token_here'

with open('/root/.claude.json', 'w') as f:
    json.dump(config, f, ensure_ascii=False, separators=(',', ':'))
```

ただし、重要な注意点がある。**この修正は「次回のClaude Codeセッション起動時」に有効になる。**

修正を加えた当日のセッション内では、MCPサーバーはすでに旧設定（プレースホルダー）で起動済みのため、設定ファイルを書き換えてもそのプロセスには届かない。修正後の最初の新しいセッション起動で、Claude CodeがMCPサーバーを再起動し、`~/.claude.json`を読み込むことで初めて正しいトークンが渡される。

## 修正確認（2026-08-17）

本章追加セッション（2026-08-17）においてGitHub MCP `create_or_update_file`の書き込みテストを実施したが、**依然として`Authentication Failed: Requires authentication`が継続**している。`~/.claude.json`への修正（2026-08-10実施）が有効になっていない可能性があり、引き続きsystem部で調査中。本章はローカルclone（`/root/zenn-content`）経由でpushした（`github_mcp_error_log.md`に記録）。

## 設計としての教訓

1. **MCPサーバーと.envは独立した設定空間**  
   MCPに必要なAPIキーは`~/.claude.json`の`env`セクションに明示する。プロジェクトの.envに書いても届かない。

2. **フォールバックを「恒久対処」と混同しない**  
   フォールバックで本番が動き続けていても、根本原因の追跡ファイルは消さない。`github_mcp_error_log.md`のような専用ファイルで「まだ解決していない」という事実を可視化し続ける。

3. **「読み取りが正常」≠「認証が正常」**  
   公開リポジトリへの読み取りアクセスは認証不要。書き込み失敗とは別の話になる。

## 次の章へ

次章以降も、直近でスキ・PVが伸びたテーマを深掘りする形で更新していく。
