---
title: "Claude Code .claude/agents/ の設計実践 — 19体のエージェントをMarkdownで管理する方法"
emoji: "🏗️"
type: "tech"
topics: ["claudecode", "ai", "llm", "automation", "agents"]
published: true
---

## 19本のMarkdownが会社を動かしている

`/root/ai-sales-company/.claude/agents/` に Markdown ファイルが 19 本ある。薬の自動販売機の営業、印刷サービスの営業、りんご贈答品の企画、競合調査、株価分析、副業収益化。この 19 体が深夜にスケジューラから起動されて、今日もどこかで動いている。

ファイル一覧がこれだ。

```bash
$ ls .claude/agents/
accea.md       audit.md        bvsync_intel.md  ceo.md
designer.md    fukugyo.md      globalinfo.md    kabutan.md
listmaking.md  marketing.md    newbiz.md        news.md
shinanoagri.md system.md       trainer.md       yakujihan.md
aiinfo.md      fukugyo_analyst.md  fukugyo_publisher.md
```

この記事では「なぜ Markdown でエージェントを管理するのか」と「どう書くと精度が上がるか」を、実際のコードを引いて説明する。

---

## エージェント MD の構造

各ファイルの先頭は YAML frontmatter だ。

```yaml
---
name: yakujihan
description: 薬の自動販売機（OTC医薬品IoT販売機・スマートメッズ24）の営業エージェント。薬局・ドラッグストア・ホテル・駅・商業施設向けのリード獲得、提案書作成、メール文作成、テレアポスクリプト、フォロー管理、Gmailログ確認、薬局アプローチ戦略立案など。「薬自販機」「OTC」「ブイシンク」「スマートメッズ」「薬局営業」「ホテル設置」「設置場所」に関する作業はこのエージェントに依頼する。
model: sonnet
---
```

frontmatter の直後に `@` で始まる行が続く。

```
@/root/ai-sales-company/output/yakujihan/HANDOFF.md
@/root/ai-sales-company/output/yakujihan/lessons_learned.md
@/root/ai-sales-company/memory/company_lessons.md
```

この `@` 参照が鍵だ。エージェントが起動するたびに、指定したファイルの内容がシステムプロンプトに自動挿入される。HANDOFF.md には「前回の進捗」、lessons_learned.md には「過去の失敗事例」が入っている。セッションをまたいでもコンテキストが引き継がれる理由がこれだ。

---

## description の書き方で精度が変わる

Claude Code は複数のエージェントが登録されているとき、`description` を読んでどのエージェントを起動するかを判断する。だから description は「何ができるか」より「どんな言葉で呼ばれるか」で書く。

**NG：**
```
薬の自動販売機の営業担当エージェント
```

**OK：**
```
薬の自動販売機（OTC医薬品IoT販売機・スマートメッズ24）の営業エージェント。
薬局・ドラッグストア・ホテル・駅・商業施設向けのリード獲得、提案書作成、
メール文作成、テレアポスクリプト、フォロー管理、Gmailログ確認、薬局アプローチ戦略立案など。
「薬自販機」「OTC」「ブイシンク」「スマートメッズ」「薬局営業」「ホテル設置」「設置場所」
に関する作業はこのエージェントに依頼する。
```

違いは 2 つある。1 つ目は**固有名詞を列挙している**こと。「スマートメッズ24」「ブイシンク」といった正式名称と略称の両方を書く。ユーザーが省略した言い方で呼んだときも正しくルーティングされる。

2 つ目は**呼び出しパターンを末尾に明示している**こと。「〇〇に関する作業はこのエージェントに依頼する」という文言が、Claude のエージェント選択ロジックに直接効く。

---

## HANDOFF.md の設計パターン

エージェントの記憶は `HANDOFF.md` に集約する。フォーマットの例：

```markdown
# 薬自販機営業エージェント HANDOFF

最終更新: 2026-06-27

## 進行中タスク
- 〇〇薬局（東京・世田谷）へのフォローメール送付 → 2026-07-01 期限

## 完了済み（今週）
- リスト100件作成 ✅
- テレアポスクリプト v3 作成 ✅

## 要確認（オーナーへ）
- 補助金の採択率について公式発表待ち

## 禁止事項メモ（重要）
- アパホテル・羽田空港はリストに含めない（複数回のNG済み）
```

3 点を徹底している。**最終更新日を冒頭に入れること**（古い情報を信じるリスクを防ぐ）、**完了済みを残すこと**（次セッションで重複作業しない）、**禁止事項を専用セクションで管理すること**（同じミスが繰り返されやすいため）。

禁止事項は本文中に埋めるより独立したセクションにしたほうが確実に機能する。実際、yakujihan.md の本文冒頭に「⚠️ 必ず守る禁止事項」セクションを設けてから、リスト混入のミスが止まった。

---

## session_end.js が自動でgit pushする

エージェントが仕事を終えたとき、HANDOFF.md が手元のファイルに残ったままだと意味がない。翌日の起動で古い状態を読むからだ。

`.claude/hooks/session_end.js` がこれを解決している。セッション終了時に自動実行され、変更があった場合だけ git commit & push する。

```js
// session_end.js（抜粋）
const diff = execSync('git diff --name-only HEAD', { cwd: REPO }).toString().trim();
const hasHandoff = diff.split('\n').some(f => f.includes('HANDOFF') || f.includes('output/'));
if (hasHandoff) {
  execSync('git add -A', { cwd: REPO });
  try {
    execSync('git diff --cached --quiet', { cwd: REPO });
    // 差分なし → commit不要
  } catch (_) {
    execSync('git commit -q -m "[auto-session-end] HANDOFF/output更新"', { cwd: REPO });
    execSync('git push origin HEAD --quiet', { cwd: REPO });
  }
}
process.exit(0); // ← これを省略するとUIがフリーズする
```

末尾の `process.exit(0)` は必須だ。省略すると Claude Code の UI がフリーズしてターミナルごと固まる。一度やった。

---

## 失敗してわかったこと

**エージェントを20体以上に増やしたら、誰に頼むか分からなくなった。**

description が重複すると Claude が迷う。「薬局」というキーワードで yakujihan と bvsync_intel（競合調査）の両方がヒットしてしまい、誉れある仕事が wrong agent に流れた。

解決策はシンプルだった。description の末尾に「〇〇に関する作業はこのエージェントに依頼する」という一文を各 MD に明示し、競合するキーワードを整理した。

**HANDOFF.md が 2000 行を超えたときも詰んだ。**

コンテキストウィンドウを圧迫し、肝心の業務指示が末尾に押し出される。対策は月次の「蒸留」だ。完了済みタスクを削除し、要点だけを残す。これを怠ると徐々に精度が落ちていく。

---

## このシステムの全実装を公開している本

`.claude/agents/` の設計・HANDOFF.md の設計パターン・hooks の実装・スケジューラとの連携を丸ごと解説した Zenn の有料本がある。

**[Claude Code で作る AI 自動化会社 完全解説](https://zenn.dev/nakayama_acari/books/ai-company-automation)**

¥4,980。架空のサンプルではなく、今動いているシステムの実コードと設定ファイルをそのまま載せている。
