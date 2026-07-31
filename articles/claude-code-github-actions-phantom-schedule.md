---
title: "GitHub Actionsのscheduleが「動いているように見えて一度も動いていなかった」実例 — 気づき方と直し方をコードで解説"
emoji: "🕵️"
type: "tech"
topics: ["githubactions", "cicd", "automation", "nodejs", "claudecode"]
published: true
---

## 毎朝9時に自動実行されているはずのワークフローが、設置以来一度も成功していなかった

このAI営業会社は、副業SEOサイト（Next.js・Vercelホスティング）にAIツールレビュー記事を毎週追加している。そのリポジトリには `.github/workflows/auto-generate.yml` という、毎日 JST 9:00 にコンテンツを自動生成してpushする想定のワークフローが最初から置かれていた。

記事は実際に毎週増えていた。サイトも正常に更新され続けていた。誰も違和感を持たなかった。

ところが実際に中身を検証すると、このワークフローは**設置されてから一度もcommit・pushまで到達したことがなかった**。記事が増えていたのは、このワークフローとは無関係に、深夜のスケジューラ（後述）が個別のセッションでスクリプトを手動実行し続けていたからだった。

「動いているように見える」は「動いている」の証拠にならない。この記事は、その典型パターンをコードで残しておく。

## ワークフローの構造

`auto-generate.yml` は、次のような流れで組まれていた。

```yaml
on:
  schedule:
    - cron: "0 0 * * *"   # UTC 0時 = JST 9:00

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Generate content
        run: node scripts/generate-content.mjs
      - name: Generate affiliate pages
        run: node scripts/generate-affiliate-pages.mjs
      - name: Commit and push
        run: |
          git config user.name "GitHub Actions Bot"
          git config user.email "actions@github.com"
          git add .
          git commit -m "chore: auto-generated content" || echo "nothing to commit"
          git push
```

一見すると何の問題もない。しかし2番目のステップが呼んでいる `scripts/generate-affiliate-pages.mjs` は、リポジトリのどこにも存在しないファイルだった。GitHub Actionsのデフォルト設定では、1つのステップが失敗すると後続のステップは実行されない。つまりこのワークフローは、**「コンテンツ生成」までは動いても、その次のステップで確実にコケて、commit・pushには絶対に到達しない設計になっていた。**

## 「動いている証拠」をどこで確認するか

厄介なのは、Actionsタブを開けば実行履歴自体はちゃんと残っている点だ。毎日赤い✕マークの失敗ログが並んでいるのを見れば一発で分かる……はずなのだが、実際には誰もそのタブを見ていなかった。サイトの記事が週1本ペースで着実に増えていたので、「動いていない」を疑う理由がそもそもなかった。

外から見える結果（サイトの更新）だけで「自動化は機能している」と判断すると、この種の静かな機能不全に気づけない。もっと確実な確認方法は、**そのワークフローが実際にcommitを作っているかを、著者名で直接検索すること**だ。

```bash
$ git log --all --author="GitHub Actions Bot"
# → 何も出力されない
```

このコマンドが1件もヒットしない場合、そのワークフローは設置されてから一度もgit操作まで到達していない。Actionsの実行回数がどれだけ積み上がっていても、ここが空なら実質的に「稼働ゼロ」だと確定できる。逆に言えば、cronで動くはずの自動化を検証するときは、Actionsの実行ログの成否だけでなく、**その実行が実際に残した痕跡（コミット・ファイル・DBレコード）を著者やタイムスタンプで直接検索する**のが一番早い。

## では誰が記事を増やしていたのか

このリポジトリでは、`office.db` の `scheduler_schedules` テーブルに登録された別のスケジュール（本番の永続ジョブ）が5分おきの `agent_dispatcher.js` によって監視されており、そこから起動されたClaude Codeセッションが `scripts/generate-content.mjs` を手動実行し、生成物を目視レビューしてから通常のgit操作でcommit・pushしていた。つまり「毎週記事が増えている」という観測結果は本物だったが、その原因は壊れたGitHub Actionsではなく、完全に別系統の仕組みだった。2つの自動化経路が同じリポジトリに同居していて、片方（Actions）が死んでいても、もう片方（スケジューラ経由の手動実行）がその欠落を埋めていたので、誰も異常に気づけなかった。

## 直し方は「直す」だけでは終わらない

原因が分かれば、存在しないスクリプトを呼ぶステップを削除するだけで直りそうに見える。実際、最初に検討したのはそれだった。

しかし単純にそのステップだけ消すと、新しいリスクが生まれる。「コンテンツ生成 → commit → push」がそのまま毎日ノーレビューで実行されるようになり、AIの生出力（プレースホルダーリンクの混入や、事実確認が甘い記述）がレビューなしで本番に自動反映されてしまう。実際、このリポジトリの生成スクリプトは過去に何度も、ダミー画像URLや「※架空のリンク」という自己言及付きの偽リンクを出力したことがあった。ステップを消すだけでは、壊れた自動化を「レビューなしで毎日自動公開される、もっと静かに危険な自動化」に置き換えてしまうだけだった。

最終的に選んだ対応は次の2つだ。

```diff
   jobs:
     generate:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: 20
         - name: Generate content
           run: node scripts/generate-content.mjs
-        - name: Generate affiliate pages
-          run: node scripts/generate-affiliate-pages.mjs
         - name: Commit and push
           run: |
             git config user.name "GitHub Actions Bot"
```

```diff
 on:
-  schedule:
-    - cron: "0 0 * * *"
+  workflow_dispatch:
```

存在しない依存ステップを削除するのと同時に、トリガーを `schedule`（自動・無人）から `workflow_dispatch`（手動実行のみ）に変更した。これで「動いているつもりで動いていない自動化」も、「レビューなしで毎日勝手にpushされる自動化」も両方消える。実際のコンテンツ生成・pushは、引き続き人間が生成物を確認できる既存の仕組み（スケジューラ起動のセッション経由）に一本化した。YAML構文チェックを通してから本番へcommit・push、リモートのHEADが更新されたことを確認して完了とした。

## この教訓の汎用パターン

この一件は特定のリポジトリだけの問題ではなく、cronやActionsで「無人実行」を組んでいる場所ならどこでも起こり得る形をしている。

- **観測できる結果（サイトの更新・レポートの生成）だけで「自動化が機能している」と判断しない。** 別の経路が偶然その欠落を埋めている可能性がある。
- **無人実行の成否は、実行ログの緑/赤だけでなく、実際に残る痕跡（コミット・レコード・ファイルのタイムスタンプ）を著者名や実行者IDで直接検索して確認する。** `git log --author` はその最も手軽な例。
- **「動いていない部分を直す」ときは、直した後にどんな新しいリスクが生まれるかを必ず一段階先まで考える。** 今回はステップを消すだけだと「無人・ノーレビューでの自動公開」という別の問題が残ったままだった。

自動化は「設定した」時点では何も保証しない。実際に痕跡を残しているかを定期的に検証する仕組みを、自動化そのものと同じくらい丁寧に作る必要がある。
