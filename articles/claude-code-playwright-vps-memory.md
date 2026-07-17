---
title: "Playwrightの自動投稿が原因不明で失敗し続けた真犯人は、自動化コードではなく`free -h`の中にいた"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "playwright", "nodejs", "automation", "linux"]
published: true
---

## 5回連続で失敗しても、コードはどこも壊れていなかった

X（旧Twitter）への自動投稿スクリプトが、3日連続で5〜7回連続の送信失敗を起こしたことがある。ログイン状態は毎回正常。投稿ボタンも押せている。それでも「投稿が届いたこと」の確認だけがすり抜ける。コードを何度読み直しても、ロジックはどこもおかしくなかった。

原因を突き止めたのは、コードではなく`free -h`だった。このAI営業会社ではPlaywrightで note.com とXへの自動投稿を行っているが、その原因調査で「自動化の不具合だと思っていたものが、実はVPSのメモリ不足だった」という切り分けに時間がかかった実例をそのままコードで残しておく。

---

## 投稿の成否は「サーバー応答」で確認している

まず前提として、このスクリプト（`fukugyo_x_post.js`）は投稿ボタンを「押した」ことではなく、Xのサーバーから返る通信（`CreateTweet`が200番で返ってくること）を見て初めて「投稿できた」と判定する設計になっている。

```javascript
// 送信：CreateTweet 200 を確認できるまで最大3回クリックを試みる
let res = null;
for (let attempt = 1; attempt <= 3 && !res; attempt++) {
  const created = page.waitForResponse(
    (r) => /CreateTweet/i.test(r.url()) && r.status() === 200,
    { timeout: 12000 }
  ).catch(() => null);

  await page.evaluate(() => {
    const b = document.querySelector('[data-testid="tweetButtonInline"]');
    if (b) b.click();
  });

  res = await created;
  if (!res) console.warn(`⚠️ CreateTweet 200 未確認（試行 ${attempt}/3、再試行します）`);
}
if (!res) throw new Error('X_POST_NOT_CONFIRMED');
```

見た目のクリックが成功していても、Reactアプリの内部処理が追いついていなければ何も送信されていない。だからサーバー応答を確認するまでリトライする設計にしていた。ここまでは実際に機能していて、二重投稿は一度も起きていない。それでも「3回リトライしても届かない」が高頻度で発生し続けていた。

---

## ログイン確認スクリプトが、5分以上応答しなくなった

ある朝、投稿前のログイン確認スクリプト（`check_x_last_post.js`。投稿はせず最新ツイートを取得するだけの読み取り専用処理）を実行したところ、通常なら数十秒で終わる処理が5分経っても進捗ゼロのまま応答しなかった。

このスクリプトが呼び出しているログイン判定は、こういうコードだ。

```javascript
// session_context.js
async function checkLoggedIn(page, { okSelectors, loginRe, timeout = 20000 }) {
  let el = null;
  try {
    el = await page.waitForSelector(okSelectors.join(', '), { timeout });
  } catch (_) { /* not found within timeout */ }
  const url = page.url();
  return { loggedIn: !!el, trulyExpired: loginRe.test(url), url };
}
```

タイムアウトは20秒しか設定していない。それなのに5分応答が返ってこないというのは、Playwrightのロジックの問題ではなく、その手前の層——ブラウザの起動やページ描画そのものが止まっている可能性が高い。ここで初めて`free -h`を叩いた。

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:          1.9Gi       1.7Gi        98Mi        12Mi       140Mi        90Mi
Swap:         4.0Gi       3.6Gi       412Mi
```

空きメモリはわずか98MB、スワップは4GB中3.6GBまで使い切っていた。このVPSは物理メモリが1.9GBしかない。Claude Codeのセッションが複数（各250〜310MB）+ MCPサーバー群が同時に走っていると、あっという間にこの状態に陥る。スワップが埋まった状態でのディスクI/Oは極端に遅く、ブラウザのページ描画がその影響をまともに受けていた。

---

## 「Playwrightのバグ」に見えていたものの正体

このメモリひっ迫状態で`fukugyo_x_post.js`を実行すると、投稿欄が見つからないエラー（`X_COMPOSER_NOT_FOUND`）が出る。ログだけ見ると「セレクタが変わった」「Xの仕様変更か」と疑いたくなる。だが実際に取得していたデバッグ画像を確認すると、原因は全く違っていた。

1回目の失敗：画面はホームタイムラインが正常に表示されていた。ログイン画面ではない。単に「描画が終わってから判定するまでのタイミング」がずれていただけの誤検知。

2回目の失敗：画面は白紙にローディングスピナーが回っているだけだった。ページの描画自体がまだ終わっていない状態で、セレクタを待つ処理がタイムアウトしていた。

どちらも「ログインが切れた」わけでも「Xの画面構造が変わった」わけでもない。スワップひっ迫によってChromiumの描画が通常より何倍も遅くなり、コード側が想定しているタイムアウトの範囲に収まらなくなっていただけだった。3回目の試行では、描画が追いついたタイミングでたまたま成功し、`CreateTweet 200`を確認できた。

---

## 再発防止：エラーが出たら、まず`free -h`

これまでこの不具合は「Playwrightのクリック処理・送信確認ロジックの恒久的な欠陥」として扱い、コード側の修正を重ねてきた（透明オーバーレイ回避のfocus()方式への変更、ボタンの有効化待ちなど）。それ自体は正しい改善だったが、根本原因の一部はコードの外にあった。

再発防止として社内ルールに加えたのは次の一行だ。

> 自動化スクリプトが原因不明のタイムアウト・誤検知を起こしたら、コードを疑う前に`free -h`でVPSのメモリ・スワップ状態を確認する。空きメモリが100MB台まで落ちていたら、他に重いセッションが同時に動いていないか確認する。

このVPSでは同時に走らせるClaude Codeセッションの上限（2本まで）も別途ルール化している。自動投稿の信頼性は、投稿スクリプトのコード品質だけでなく、同じVPS上で他に何が動いているかにも左右される——これは実際にログとメモリの数字を突き合わせるまで気づけなかった。

---

## この会社のPlaywright自動化・失敗の全記録

headlessでbot判定される問題の回避、ログイン切れの誤診断防止、透明オーバーレイのクリック回避、そして今回のメモリひっ迫まで、note/X自動投稿の実装を「うまくいった話」だけでなく「今も直している最中の話」としてすべてコード付きでZennの有料本にまとめている。

**[Claude Code で作る AI 自動化会社 完全解説](https://zenn.dev/nakayama_acari/books/ai-company-automation)**（¥4,980）

架空の解説ではなく、今このVPSで実際に動いている（そして時々止まる）コードをそのまま載せている。
