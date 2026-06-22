---
title: "Claude Code と VPS で16エージェントのAI会社を作った話"
emoji: "🤖"
type: "idea"
topics: ["claudecode", "ai", "automation", "vps", "pm2"]
published: true
---

## 3商材を16体のAIで回している

薬の自動販売機、印刷サービス、高級りんご贈答品。この3つをターゲットの違う別々の営業チームが担当している、小さな営業会社があります。

各チームにはリード獲得・提案書作成・メール文生成・市場調査が必要です。3商材 × 4タスク = 12種類の繰り返し業務。これを現在、**16体の AI エージェントが分担して動かしています。**

バックエンドは Hetzner の VPS 1台（$15/月）、PM2 で5プロセスを常駐、Claude Code が脳として機能しています。

```bash
$ pm2 list

┌────┬──────────────────────┬────────┬───────────┐
│ id │ name                 │ uptime │ status    │
├────┼──────────────────────┼────────┼───────────┤
│  4 │ accea-crm            │ 27m    │ online    │
│  0 │ ai-office            │ 95s    │ online    │
│  1 │ cloudflare-tunnel    │ 6D     │ online    │
│  3 │ office-tunnel        │ 5D     │ online    │
│  2 │ vsync-crm            │ 71m    │ online    │
└────┴──────────────────────┴────────┴───────────┘
```

これは架空の図解ではありません。今この瞬間動いているプロセスです。

---

## エージェントは「社員」として設計されている

各エージェントは `.claude/agents/` 配下の Markdown ファイルで定義されています。

```markdown
---
name: yakujihan
description: OTC医薬品IoT販売機の営業エージェント。薬局・ホテル・商業施設向けの
  リード獲得、提案書作成、メール文作成を担当。
model: sonnet
---

@/root/ai-sales-company/output/yakujihan/HANDOFF.md
@/root/ai-sales-company/output/yakujihan/lessons_learned.md

# 薬の自動販売機 営業エージェント

## アイデンティティ
OTC医薬品自販機（スマートメッズ24）の凄腕営業AI。
薬局相手には：制度加算・コスト削減・24時間対応の観点で話す。
ホテル相手には：インバウンド対応・スタッフ不要・収益多様化の観点で話す。
```

`@` で始まる行は**ファイル参照**です。エージェントが起動するたびに、前回の引き継ぎ情報（HANDOFF.md）と過去の失敗事例（lessons_learned.md）を自動読み込みします。これにより、セッションをまたいだ継続性が生まれます。

---

## フックシステムが自律性を支える

Claude Code の `settings.json` にはフック機能があります。セッションの開始・終了・ツール使用後などのタイミングで、任意のスクリプトを実行できます。

```json
"hooks": {
  "SessionEnd": [{
    "matcher": "",
    "hooks": [{
      "type": "command",
      "command": "node .claude/hooks/session_end.js"
    }]
  }]
}
```

`session_end.js` は、セッション終了時に HANDOFF.md の差分を検出して自動 git push します。翌朝起動したとき、昨夜の作業が GitHub に記録されています。

クラッシュリカバリも自動化されています。セッションが正常終了しなかった場合、次の起動時に直前30件のツールログとチェックポイントが自動注入されます。「続きお願い」の一言で前回の文脈が復元される仕組みです。

---

## 30+ MCP サーバーが実作業をこなす

MCP（Model Context Protocol）は、外部サービスを Claude のツールとして接続する仕組みです。このシステムでは 30 以上の MCP サーバーが動いています。

主な用途:

- **Playwright MCP**: ブラウザ自動操作で競合サイトのデータ収集
- **Firecrawl MCP**: Webスクレイピングで営業先リスト作成
- **Google Sheets MCP**: 営業データの自動集計・更新
- **PowerPoint MCP**: 提案書の .pptx ファイル自動生成
- **hourei MCP**: 改正薬機法（2027年施行）の条文を即参照

「薬局100件のリスト作って」と yakujihan エージェントに頼むと、Firecrawl と OSM（OpenStreetMap）MCP を組み合わせて薬局データを収集し、Excel ファイルとして `output/yakujihan/` に保存します。

---

## 何がうまくいって、何が失敗だったか

**うまくいったこと:**
- CLAUDE.md を「会社憲法」として設計したこと。全エージェントの行動指針・保存先・禁止事項を一箇所に集約した
- フックで教訓を自動記録する仕組み。「NG」という言葉を検知して教訓ファイルに追記するフックが地味に効いている

**失敗したこと:**
- エージェント数を増やしすぎた時期があった。20超えたときに「誰に頼めばいいか分からない」という問題が発生した
- HANDOFF.md が数週間で数千行になる。定期的に蒸留しないとコンテキストを圧迫する
- CRM を後から自作するはめになった。最初から設計しておくべきだった

---

## これを詳しく解説する本を書いています

このシステムの設計・実装・運用を丸ごと解説する Zenn の有料本を執筆中です。

**[Claude Code で作る AI 自動化会社 完全解説](https://zenn.dev/nakayama_acari/books/ai-company-automation)**

価格: ¥4,980

扱う内容:
- エージェント定義ファイルの設計（description の書き方で精度が変わる）
- HANDOFF.md と lessons_learned.md の設計パターン
- フックシステムの実装（crash_state.json によるクラッシュ検知）
- MCP サーバーの選択・設定・組み合わせ
- CRM を Node.js + SQLite で自作した設計判断
- 深夜自律稼働モードの設計（スケジューラと権限管理）
- 実際のコスト（VPS $15/月、Claude API 従量課金の実績値）

「ChatGPT でできる」ではなく「Claude Code の CLI を軸に会社インフラを組んだ」記録です。同じ構成を再現したいエンジニアのための本です。
