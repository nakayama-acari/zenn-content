---
title: "第5章: Playwright自動化 — note/X自動投稿の実装、そして今も終わっていない失敗"
---

## この章で学ぶこと

- headless環境でX/noteに自動ログインすると「bot判定」されて詰む問題と、その回避策
- ログイン切れを「本当に切れた」場合と「一時的な描画遅延」に区別する判定ロジック
- 透明オーバーレイにクリックを吸われて自動投稿が失敗する実際の不具合と修正
- **今この瞬間も解決していない、3日連続で投稿確認が失敗し続けている実話**

## 実際のシステム構成

### headlessでは弾かれる。だからheadedをxvfb配下で動かす

このシリーズを最初に読んだ人は「AIが勝手に投稿している」と聞くと、裏側は真っ黒い画面でスクリプトが走っているだけだと想像すると思う。実際は違う。VPSにはディスプレイが無いので、`xvfb-run`という「仮想ディスプレイ」を使い、その中で**本物のブラウザ画面を持ったChromium**を起動している。

```javascript
// session_context.js
function ensureDisplay() {
  if (process.env.DISPLAY) return;
  const r = spawnSync('xvfb-run', ['-a', 'node', ...process.argv.slice(1)], {
    stdio: 'inherit',
    env: process.env,
  });
  process.exit(r.status == null ? 1 : r.status);
}
```

理由は単純で、headless（画面を持たないモード）でX/noteにアクセスすると、cookieが有効でもbot判定されて投稿欄そのものが出てこないことが実際にあった。「セッションが切れた」と誤診断して延々と再ログインを試みる無駄が発生していた。画面を持たせるだけで、この誤判定が消えた。

### storageStateの複製ではなく「永続プロファイル」を使う

もう一つの工夫は、cookieをJSONで持ち回る一般的な方式（`storageState`）をやめたことだ。

```javascript
async function openAuthedContext({ profileDir, legacyPath, userAgent }) {
  const seeded = fs.existsSync(path.join(profileDir, 'Default'));
  fs.mkdirSync(profileDir, { recursive: true });

  const context = await chromium.launchPersistentContext(profileDir, {
    headless: false,
    viewport: { width: 1280, height: 900 },
    userAgent: userAgent || 'Mozilla/5.0 (Macintosh; ...) Chrome/120.0.0.0 Safari/537.36',
    args: ['--disable-blink-features=AutomationControlled'],
  });

  // 初回のみ既存の有効cookieをseed（2回目以降はプロファイルが自前で保持）
  if (!seeded && legacyPath && fs.existsSync(legacyPath)) {
    const state = JSON.parse(fs.readFileSync(legacyPath, 'utf8'));
    if (state?.cookies?.length) await context.addCookies(state.cookies);
  }
  return context;
}
```

`launchPersistentContext`は、ブラウザの「プロファイルフォルダ」ごとディスクに保存する方式。cookieだけでなくlocalStorage・キャッシュ・拡張機能の状態まで全部保持されるので、Chromeを普段使いで閉じて開いても情報が消えないのと同じ理屈が働く。初回だけ、これまで保存していたcookie（`x_session.json`）を注入し、2回目以降はプロファイル自体が記憶を持ち続ける。

### 「本当に切れた」か「一時的な遅延」かを区別する

セッション判定でもっとも消耗したのがここだった。

```javascript
async function checkLoggedIn(page, { okSelectors, loginRe, timeout = 20000 }) {
  let el = null;
  try {
    el = await page.waitForSelector(okSelectors.join(', '), { timeout });
  } catch (_) { /* not found within timeout */ }
  const url = page.url();
  return { loggedIn: !!el, trulyExpired: loginRe.test(url), url };
}
```

「ログイン済みを示す要素が見つからない」＝「セッション切れ」と決めつけていた頃は、単にタイムライン描画が遅かっただけのケースまで「セッション切れ」としてSlackに誤報していた。今は「ログイン要素が見えるか」と「URLがログイン画面に飛ばされているか」の2軸で判定し、後者が真のときだけ「本当に切れた」と扱う。前者だけなら「一時的な遅延」として処理を止め、様子を見る。

## なぜこの設計にしたか

**失敗した設計案1：headlessのまま、cookieだけ都度読み込む**
→ VPSのIPからのheadlessアクセスはX側のbot検知に引っかかりやすく、cookieが生きていても投稿欄自体が描画されないことがあった。「動くときは動くが、動かないときの原因が全く分からない」状態が続いた。

**失敗した設計案2：「ログイン要素が見えない＝即セッション切れ」という単純判定**
→ 描画が少し遅いだけの正常な状態を「セッション切れ」と誤診断し、実際には有効なセッションに対して不要な再ログイン依頼をSlackに送り続けた。URLベースの判定を足すことで「本当に切れているか」を切り分けられるようになった。

**失敗した設計案3（現在進行形）：投稿ボタンを押したら成功、という前提**
→ これは後述するが、まだ「失敗した設計案」のまま解決していない。

## 実装のポイント

### 透明な板にクリックを吸われる不具合

X画面の右下には、Grok用チャットの引き出しパネルがある。これを開閉するための透明な板（オーバーレイ）が画面全体を覆っていることがあり、投稿欄やボタンへの「クリック」がその板に吸われて反応しなくなる現象が実際に起きた。

```javascript
// 画面右下のGrok/Chatドロワー用の透明オーバーレイがマウスクリックを吸ってしまい
// inputBox.click() がタイムアウトすることがある。
// ヒットテストを経由しない focus() + キーボード入力で同じ問題を回避する。
await inputBox.evaluate((el) => el.focus());
await page.waitForTimeout(200);
await page.keyboard.type(text, { delay: 20 });
```

「クリックする」のではなく「フォーカスを当てて、キーボードで文字を打つ」方式に変えることで、透明な板を経由せずに直接テキストが入力できるようになった。noteのアイキャッチ画像アップロードボタンも同様に、`aria-label="画像を追加"`という属性で直接要素を特定し、見た目のクリック位置に頼らない実装にしている。

### ボタンが「押せる状態」になるのを待つ

```javascript
// 投稿ボタンが「有効化」されるまで待つ。Reactの状態反映前にクリックすると空振りし、
// CreateTweetが発火せずX_POST_NOT_CONFIRMEDになる。
await page.waitForFunction(() => {
  const b = document.querySelector('[data-testid="tweetButtonInline"]');
  return !!b && b.getAttribute('aria-disabled') !== 'true';
}, { timeout: 8000 }).catch(() => {});
```

投稿ボタンは、文字を打った直後はまだ「無効」状態（`aria-disabled="true"`）になっていることがある。ここでクリックすると、見た目は押せているのに実際には何も起きない。ボタンの状態が「有効」に変わるのを待ってからクリックする、という一手間を入れて改善を試みた。

### それでも解決していない：3日連続の送信未確認

ここからは、あえて「うまくいった話」ではなく「今日も動いていない」ことをそのまま書く。この記事を書いている時点で、投稿の成否を確認する処理が3日連続で高頻度に失敗している。

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
}
if (!res) throw new Error('X_POST_NOT_CONFIRMED');
```

このスクリプトは、投稿の成功を「サーバーからの応答（`CreateTweet`という通信が200番で返ってくること）」で確認している。ボタンを押した「つもり」ではなく、実際にサーバーに届いたかを確認する設計自体は間違っていないはずだった。

ところが直近3日間（7/10・7/12・7/13）、この確認が5〜7回連続で失敗する現象が繰り返し起きている。ログイン状態は毎回正常。投稿ボタンも押せている。それでも「届いたこと」の確認だけがすり抜ける。二重投稿を避けるために、失敗のたびに別スクリプトで「本当に投稿されていないか」を確認してから再試行しているが、確認と再試行を5回繰り返して打ち切る日が3日続いている。

原因はまだ特定できていない。この章を「完成された仕組みの説明」として終わらせず、「今もログを見ながら追いかけている最中の不具合」として書いているのは、実際にそういう状態だからだ。

## この章のまとめ

| 課題 | 対応状況 |
|------|------|
| headlessでbot判定される | 解決済み（xvfb＋headed Chromiumの永続プロファイル） |
| セッション切れの誤診断 | 解決済み（ログイン要素＋URLの二軸判定） |
| 透明オーバーレイにクリックを吸われる | 解決済み（focus()＋keyboard.type()、DOM click方式） |
| 投稿の成功確認が高頻度で失敗する | **未解決（3日連続で再現中）** |

自動化の記事は「うまくいった話」として書かれることが多い。だがこのシステムの実態は、直しては別の場所で壊れ、直しては別の場所で壊れることの繰り返しだ。今回は最後まで解決を報告できないまま章を閉じる。

## 次の章へ

第6章では、このシステムが実際に使っている30以上のMCPサーバー（GitHub・Playwright・Slack・Google Drive・検索系など）の使い分けと、ツールが繋がりきっていない時にどう振る舞うかを解説する。
