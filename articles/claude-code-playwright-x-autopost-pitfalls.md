---
title: "Playwright で X 自動投稿を3ヶ月運用してわかった落とし穴と解決策"
emoji: "🐦"
type: "tech"
topics: ["playwright", "nodejs", "twitter", "自動化", "claudecode"]
published: true
---

深夜3時に起動したスクリプトが、毎回4〜5回失敗してから6回目でやっと成功する。それが3ヶ月続いた。

自動投稿なのに「毎回人間が見ていないと正常に終わらない」では本末転倒だ。
この記事は、**Playwright を使ってX（旧Twitter）へ自動投稿するシステムを構築・運用する中で実際に踏んだ落とし穴と、その解決策のコード**をまとめたものだ。

---

## 構成の概要

使用技術：
- Node.js + Playwright（`playwright` パッケージ）
- VPS（Ubuntu 26.04・RAM 1.9GB）
- PM2（常駐プロセス管理）
- Claude Code（スクリプトの実行制御）

基本的な投稿フローは以下の通り：

```
1. Playwrightでブラウザを起動（永続セッション使用）
2. x.com を開き、セッションを復元
3. 投稿欄にテキストを入力
4. 「ポスト」ボタンをクリック
5. CreateTweet レスポンスを監視して成否を判定
6. 成功・失敗をSlackに通知
```

---

## 落とし穴1：「投稿した」か「していない」かが判定できない

最初にハマったのは、投稿後の成否確認だ。

ボタンをクリックした後、**本当に投稿されたのか、クリックが空振りしたのか**がスクリプト内で判定できない。

Xはボタンをクリックすると内部的に`CreateTweet`というAPIを呼ぶ。これをPlaywrightのnetworkリスナーで監視すればよい。

```js
// ネットワークリクエストを監視（先に登録してからクリックする）
const postResponsePromise = page.waitForResponse(
  res => res.url().includes('CreateTweet') && res.status() === 200,
  { timeout: 15000 }
);

// ボタンをJavaScript経由でクリック
await page.evaluate(() => {
  document.querySelector('[data-testid="tweetButtonInline"]').click();
});

// 15秒以内にCreateTweet 200が来たら成功
const response = await postResponsePromise.catch(() => null);
if (!response) {
  throw new Error('X_POST_NOT_CONFIRMED');
}
```

ポイントは `waitForResponse` と `click()` の順序だ。**先にリスナーを登録してからクリックする**。逆にすると、クリックからリスナー登録の間に200が来てしまい、永遠に待ち続ける。

---

## 落とし穴2：セッション切れと描画遅延を混同する

ログイン状態の確認で `loggedIn: false` が返ってきた。「セッション切れだ」と思ってスキップした。
でも翌日見ると、**実際にはログインしたままで投稿も未実施**だった。

原因は描画遅延だった。Xのタイムラインが完全に読み込まれる前にセレクタを探しに行くと、ログインUI要素が未表示の状態で「ログアウト状態」と誤判定される。

```js
async function checkLoggedIn(page) {
  try {
    // 投稿ボタンが存在するかで判定（ログイン後のみ表示される要素）
    await page.waitForSelector('[data-testid="SideNav_NewTweet_Button"]', {
      timeout: 20000  // タイムアウトを20秒に延ばす
    });
    return { loggedIn: true };
  } catch {
    // タイムアウト = ログイン画面の可能性があるが、断定しない
    const snapshot = await page.content();
    const trulyExpired = snapshot.includes('ログインして')
      || snapshot.includes('Sign in to X');
    return { loggedIn: !trulyExpired, trulyExpired };
  }
}
```

重要なのは、**タイムアウト ≠ セッション切れ** という判断だ。タイムアウトしたら描画遅延の可能性を疑い、ページのHTMLを確認してから「本当にログイン画面が出ているか」を判定する。

---

## 落とし穴3：透明なオーバーレイがクリックをブロックする

ある時期から急に `Timeout: subtree intercepts pointer events` というエラーが出るようになった。

XはGrok（AIアシスタント）のドロワーUI用に、画面全体を覆う透明なオーバーレイ要素（`data-testid="mask"`）を持っている。これが**ポインターイベントを横取りして、下にある要素へのクリックを無効にする**。

通常のPlaywright `element.click()` はブラウザの座標クリックなので、このオーバーレイに当たってしまう。

解決策はDOM APIを直接呼ぶことだ：

```js
// NG: Playwright の座標クリック（オーバーレイにブロックされる）
await buttonElement.click();

// OK: JavaScript経由でDOM直接クリック（オーバーレイを無視できる）
await page.evaluate((selector) => {
  document.querySelector(selector).click();
}, '[data-testid="tweetButtonInline"]');
```

テキスト入力欄も同様の問題が起きる。

```js
// NG: 座標クリックでフォーカス
await inputBox.click();
await page.keyboard.type(text);

// OK: focus()→keyboard.type()でオーバーレイを回避
await page.evaluate(() => {
  document.querySelector('[data-testid="tweetTextarea_0"]').focus();
});
await page.keyboard.type(text);
```

---

## 落とし穴4：VPSのメモリ不足でPlaywrightが起動しない

RAM 1.9GBのVPSでClaude Codeを複数セッション動かしながらPlaywrightも起動しようとすると、ブラウザのプロセス生成自体が止まることがある。

```
Error: launchPersistentContext: Target page, context or browser has been closed
```

このエラーが出たとき、最初は「セッションファイルが壊れた」と疑った。でも本当の原因は`free -h`で確認できる：

```
$ free -h
              total   used    free
Mem:          1.9Gi  1.8Gi   94Mi
Swap:         8.0Gi  5.4Gi   2.6Gi
```

空きメモリ100MB台・swap使用量5GB超の状態では、Playwrightのブラウザプロセスが起動完了しない。

対処法は「時間を置いて再実行すること」だ。スクリプトに事前チェックを入れると検知できる：

```js
const { execSync } = require('child_process');

function checkMemoryAvailable(minMb = 300) {
  const output = execSync("free -m | awk 'NR==2{print $7}'").toString().trim();
  const availableMb = parseInt(output);
  if (availableMb < minMb) {
    throw new Error(`メモリ不足: 空き${availableMb}MB（最低${minMb}MB必要）`);
  }
  return availableMb;
}
```

---

## 落とし穴5：コードにドット記法を使うとXが自動リンク化する

コード系のツイートで技術的な内容を含めるとき、これに気づかずにやらかした。

Xは `単語.単語` のパターンをドメイン名として自動リンク化する仕様がある。`.click`・`.dev`・`.app`・`.io` のように一般的なTLDに似た文字列が続くと誤検知されやすい。

投稿後に最新ツイートを取得して確認するスクリプトを持っておくと気づける：

```js
async function checkLastPost(page, handle) {
  await page.goto(`https://x.com/${handle}`);
  await page.waitForSelector('[data-testid="tweet"]', { timeout: 10000 });
  const tweets = await page.$$('[data-testid="tweet"]');
  if (!tweets.length) return null;
  const latestText = await tweets[0].$eval(
    '[data-testid="tweetText"]',
    el => el.innerText
  );
  return latestText;
}
```

---

## 全体の教訓：失敗の切り分け順序

3ヶ月運用して最も重要だとわかったことは**「失敗した原因の切り分け順序」**だ。

```
1. 二重投稿チェック: まず確認スクリプトで最新投稿を取得する
2. セッション確認: HTMLを見て「本当にログイン画面か」を判定
3. メモリ確認: free -h で空きメモリを確認
4. リトライ上限: 5回を超えてリトライしない（諦めてファイル保存して終了）
```

特に**「二重投稿の確認なしに再試行しない」**は絶対的なルールだ。失敗したように見えて実は成功している、というパターンが一番多い。

---

## まとめ

| 問題 | 対処 |
|------|------|
| 成否判定ができない | `waitForResponse('CreateTweet')` を先に登録 |
| セッション切れ誤検知 | HTMLを見て「本当にログイン画面か」を確認 |
| クリックがブロックされる | `page.evaluate()`でDOM直接クリック |
| Playwrightが起動しない | `free -h`でメモリを確認してから実行 |
| 自動リンク誤変換 | 投稿後に確認スクリプトで内容を検証 |

Playwright・Xの仕様は更新されるため、実装時は最新のDOM構造を確認することを推奨する。

---

*「AIで副業を自動化する実況中継」は note でも毎週公開しています→ https://note.com/large_robin2291*
