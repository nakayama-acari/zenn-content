---
title: "GitHubにpush成功でもZennに記事が公開されない理由と確認スクリプト"
emoji: "🔴"
type: "tech"
topics: ["claudecode", "zenn", "github", "playwright", "automation"]
published: true
---

## `git push`が成功しているのに、記事が読めない

このAI営業会社のZennリポジトリでは、毎週新しい記事や本の章をGitHubへpushして公開している。

ある週から「さっき上げたやつ、まだ404が返ってくる」という状態が続いた。`git push origin main`は毎回正常終了していた。`git log`にも最新のコミットが残っている。GitHubのリポジトリ画面を開けば、最新ファイルがちゃんとそこにある。

それでも記事のURLを開くと404。

原因が分かるまでに20日かかった。その間に7回pushした。全部「成功」として処理された。全部「公開」に届いていなかった。

---

## Zennのデプロイが「2段階」になっている理由

Zennのリポジトリ連携は、GitHubへのpushだけでは完結しない。

フローはこうなっている：

```
① git push origin main（GitHubにファイルを送る）
          ↓
② Zennがwebhookを受け取り、デプロイキューに積む
          ↓
③ Zennがデプロイを実行し、記事・本を公開する
```

①は手元のターミナルで即座に確認できる。`Everything up-to-date` や `main -> main` という出力が返ってくれば成功だ。

問題は②と③の間に起きる。Zennのデプロイキューにはエラーチェックがあり、条件を満たさない場合は「デプロイ中断」になる。このとき、**GitHubへのpushはすでに成功している**から、`git push`コマンドの出力には何も表示されない。Zennからのメール通知もない。Slackへのアラートもない。

要するに、どこにもエラーが出ない。

---

## silent failが起きる3つのパターン

Zennのデプロイが「デプロイ中断」になるパターンは主に3つある。

### 1. slug重複

同じslug名のファイルが2つ以上存在するとデプロイが止まる。自動生成でslugを作るシステムを組んでいると特に起きやすい。

今回の事例がこれだった。あるタイミングで同じ題材を2つの異なるファイルに書いてしまい、slug(`claude-code-hooks-guide`系)が重複した状態でpushしていた。

### 2. レート制限

短時間に複数のpushをかけると、先着順でデプロイされ、残りは無視される。深夜のスケジューラが複数の章を連続してpushしたとき、後続ファイルが処理待ちのまま放置されることがある。

### 3. カバー画像の設定不備

有料本（books/配下）のconfig.yamlでカバー画像が未設定またはパス指定が間違っていると、本全体のデプロイが止まることがある。本の章を何章追加しても、親のconfigが壊れていると全部公開されない。

この3つの共通点は「GitHubへのpushは成功する」こと、そして「Zennが黙って止まる」ことだ。

---

## 20日間で何が起きていたか

後から調べてわかったことを書く。

slug重複で止まっていた1本のデプロイが、後続の全更新をブロックしていた。Zennはキュー形式で処理するため、1件が「デプロイ中断」になると後ろに続く更新も待ちの状態になる。

最初のsilent failが起きてから20日間：

- 7回pushした
- 全部「git push成功」と表示された
- Zennの記事は20日前のバージョンのままだった
- その間、AIのスケジューラは「push成功→デプロイ完了のはず」と報告し続けた

発覚した理由は偶然だった。`https://zenn.dev/dashboard/deploys` を初めてちゃんと開いたとき、最上部に「デプロイ中断 — slug重複があります」と書いてあった。

修正は数秒で終わった。重複していたファイルを削除して再pushしたら、7回分のpushが一気に処理された。20日ぶりに記事が公開された。

---

## `dashboard/deploys` が唯一の真実を教えてくれる場所

この画面には、直近のデプロイ結果が新しい順に並んでいる。

- `デプロイ成功` — 公開済み。記事は読める
- `デプロイ中断` — 止まっている。記事は読めない

この画面だけが現実を教えてくれる。`git log`でも`git status`でもGitHubのリポジトリ画面でも、ここに表示される結果は分からない。

---

## Playwrightで確認スクリプトを作った

毎回手動で`dashboard/deploys`を開くのは漏れが出る。そこで、pushのたびに自動確認するスクリプトを作った。

実際に動いているコードがこれだ（`check_zenn_deploy.js`）：

```javascript
/**
 * check_zenn_deploy.js
 * Zennのdashboard/deploysをPlaywrightで読み取り、
 * 最新デプロイが「デプロイ成功」かを確認する。
 */
const { chromium } = require('playwright');
const fs = require('fs');
const path = require('path');

const SESSION_PATH = path.join(__dirname, '../session/zenn_session.json');

async function main() {
  if (!fs.existsSync(SESSION_PATH)) {
    console.log(JSON.stringify({ ok: false, reason: 'NO_SESSION_FILE' }));
    return;
  }

  let browser = null;
  try {
    browser = await chromium.launch({ headless: true });
    const context = await browser.newContext({
      storageState: SESSION_PATH,
      userAgent: 'Mozilla/5.0 ...'
    });
    const page = await context.newPage();

    await page.goto('https://zenn.dev/dashboard/deploys', { 
      waitUntil: 'networkidle', 
      timeout: 45000 
    });

    // ログイン切れ判定
    if (page.url().includes('/enter')) {
      console.log(JSON.stringify({ ok: false, reason: 'SESSION_EXPIRED' }));
      return;
    }

    // 先頭3000文字にデプロイ結果が表示される
    const bodyText = await page.locator('body').innerText();
    console.log(JSON.stringify({ 
      ok: true, 
      excerpt: bodyText.slice(0, 3000)
    }));
  } finally {
    if (browser) await browser.close().catch(() => {});
  }
}

main().then(() => process.exit(0));
```

`excerpt`の中に「デプロイ成功」が含まれれば問題なし。「デプロイ中断」があれば止まっている。

実際には、この出力をpushスクリプトの後に自動実行し、「デプロイ成功」を確認してからSlackに「公開完了」通知を送るフローにした。

---

## 「push成功 = 公開済み」と思い込む理由

このsilent failが20日間見過ごされた原因を振り返ると、思い込みの構造がある。

`git push`は、GitHubのリモートリポジトリへの転送を確認する。その転送は100%成功していた。問題は「転送先のGitHub」が処理したあとに起きていたことで、pushコマンドのスコープ外だった。

エラーが出ない理由もここにある。`git push`は「GitHubに届いたか」しか確認しない。「Zennがデプロイしたか」は確認しない。Zennは「デプロイ中断しました」という通知を送ってこない（少なくとも今のところは）。

だから、誰も気づかない。

この構造は、CI/CDパイプラインの途中で失敗が起きるのに下流に通知しないケースと全く同じだ。GitHubへのpushはCI（継続的インテグレーション）のステップの一つで、Zennのデプロイ処理はCD（継続的デリバリー）のステップにあたる。CIが通過してもCDが止まれば公開されない。

Zennの場合、このCDの結果を取得するAPIが公開されていないため、Playwrightでの画面スクレイピングが唯一の確認手段になっている。

---

## まとめ：pushした後の確認チェックリスト

Zennのリポジトリ連携を使っている場合、pushだけで安心しないほうがいい。

```
git push origin main
  → ✅ pushが成功したことを確認（ここまでは自動）
  → 🔍 dashboard/deploys を確認（手動 or スクリプト）
    → 「デプロイ成功」→ 公開済み
    → 「デプロイ中断」→ slug重複・レート制限・config不備のどれかを疑う
```

silent failの一番の怖さは「動いているつもりで止まっていること」だ。20日間、システムは「問題なし」と報告し続けた。Zennの記事は読めないまま放置されていた。

`dashboard/deploys`を確認するスクリプト1本で、この「つもり」を潰せる。

Zennでリポジトリ連携して記事管理をしているなら、pushのたびにこの画面を確認する習慣（またはスクリプト）を持っておくのが確実だ。

---

*この記事で使っているPlaywrightスクリプトの全コード（エラーハンドリング・スクリーンショット保存・Slack通知連携込み）は、Zennの有料本「人間の工数ゼロで動くAI営業会社を作った話」で公開しています。*
