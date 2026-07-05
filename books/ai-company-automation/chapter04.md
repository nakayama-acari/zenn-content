---
title: "第4章: スケジューラ完全解説 — 誰も見ていない深夜に、なぜAIは動き出すのか"
---

## この章で学ぶこと

- cronから5分おきに叩かれる `agent_dispatcher.js` が実際に何をしているか
- 二重起動を防ぐロックファイルの実装
- daily/weekly/monthlyの「次回実行時刻」をどう計算しているか
- エージェントをヘッドレスで起動する `spawn` の実際の呼び出し方

## 実際のシステム構成

### cron設定（実際の crontab）

```
*/5 * * * * /usr/bin/node /root/ai-sales-company/scripts/agent_dispatcher.js >> /root/ai-sales-company/output/system/scheduler_dispatcher.log 2>&1
```

5分ごとに起動して「今が実行時刻を過ぎているスケジュール」がないかDBを見に行くだけの、単純な仕組みだ。スケジュール自体はcrontabに書かない。`office.db` の `scheduler_schedules` テーブルで一元管理し、cronは「5分おきに見に行く係」に徹する。

```sql
-- 実際のテーブルの中身（一部抜粋）
-- id | dept_id           | repeat_type | repeat_day | scheduled_at      | is_enabled
15   | fukugyo_analyst    | weekly      | 1          | 2026-07-13T07:00  | 1
16   | fukugyo_publisher  | weekly      | 1          | 2026-07-13T07:00  | 1
21   | fukugyo_publisher  | daily       |            | 2026-07-07T06:00  | 1
23   | fukugyo_publisher  | monthly     | 1          | 2026-08-01T09:00  | 1
```

`dept_id` がそのままエージェント名になる。`repeat_day` はweeklyなら曜日番号（0=日〜6=土）、monthlyなら日付。

### 二重起動防止のロック（実際の `agent_dispatcher.js`）

```javascript
const LOCK_PATH = '/tmp/agent_dispatcher.lock';
if (fs.existsSync(LOCK_PATH)) {
  const lockAge = Date.now() - fs.statSync(LOCK_PATH).mtimeMs;
  if (lockAge < 4 * 60 * 1000) { // 4分以内のロックは有効（cron間隔5分以内）
    process.stdout.write(`[${new Date().toISOString()}] already running (lock age ${Math.round(lockAge/1000)}s), skip\n`);
    process.exit(0);
  }
}
fs.writeFileSync(LOCK_PATH, String(process.pid));
process.on('exit', () => { try { fs.unlinkSync(LOCK_PATH); } catch (_) {} });
```

### 該当スケジュールの抽出とエージェント起動

```javascript
const dueRows = db.prepare(`
  SELECT * FROM scheduler_schedules
  WHERE is_enabled=1 AND is_done=0 AND scheduled_at <= ?
  ORDER BY scheduled_at ASC
`).all(nowStr);

// ...

const fullInstruction = `深夜自律稼働モード（スケジューラ自動実行 id=${row.id}）: ${row.instruction}`;

const child = spawn(CLAUDE_BIN, ['-p', fullInstruction, '--agent', row.dept_id], {
  cwd: ROOT,
  detached: true,
  stdio: ['ignore', agentLogFd, agentLogFd],
  env: { ...process.env, HOME: '/root', IS_SANDBOX: '1' }
});
child.unref();
```

このセッションの冒頭に必ず出てくる「深夜自律稼働モード（スケジューラ自動実行 id=16）:」という一文は、オーナーが書いたものではない。この`fullInstruction`の組み立てコードがその場で生成している。

## なぜこの設計にしたか

**問題1: cronだけで曜日・月次の複雑なスケジュールを管理すると、設定が読めなくなる**

cronの `*/5 * * * *` のような書式を19エージェント分×複数スケジュール書くと、何が何時に動くのか人間には追えなくなる。ダッシュボードのDBテーブルにして、cronは「見に行くだけ」の薄い層にした。

**失敗した設計案: cron自体に個別スケジュールを直接書く**
→ 動くには動くが、スケジュールの追加・削除のたびにcrontabを手編集する必要があり、GUIのダッシュボードから触れない。実際に一度、この構成のときに`office.db`側のレコードだけ消失するインシデントが起きている（原因不明・6/24）。crontab側は無事だったため気づくのが遅れた。今はスケジュール変更のたびに`SELECT COUNT(*) FROM scheduler_schedules`で全件確認する運用にしている。

**問題2: 同じスケジュールが二重発火するとエージェントが二重に動く**

cronの間隔（5分）より`agent_dispatcher.js`の実行時間が長くなった場合、前回の実行が終わる前に次のcronが起動する可能性がある。ロックファイルの有効期限を「4分」にしているのは、cronの実行間隔（5分）より短く設定することで「前回の起動がまだ生きているのに新しい起動が上書きする」事故を防ぐためだ。

**問題3: `-p`と`IS_SANDBOX=1`を両方指定しないと、cron配下からの起動そのものが失敗する**

```javascript
// -p: ヘッドレス(非対話)必須。IS_SANDBOX=1: root実行でのbypassPermissions許可。
// 両方欠けるとcron/spawn配下で起動失敗する
```

このコメントは実際に起動失敗を経験したあとに追記されたものだ。対話モードのままcron起動すると標準入力を待ち続けて固まる。rootでの`bypassPermissions`も明示しないと権限確認プロンプトがヘッドレス環境で永久に止まる。

## 実装のポイント

### 1. repeat_typeごとの「次回実行時刻」計算

```javascript
switch (row.repeat_type) {
  case 'daily':
    next.setDate(now.getDate() + 1);
    next.setHours(H, M, 0, 0);
    break;
  case 'weekly': {
    const targetDay = row.repeat_day; // 0=Sun..6=Sat
    const curDay = now.getDay();
    const daysUntil = ((targetDay - curDay + 7) % 7) || 7; // 最低1日先
    next.setDate(now.getDate() + daysUntil);
    next.setHours(H, M, 0, 0);
    break;
  }
  case 'monthly': {
    next.setMonth(now.getMonth() + 1);
    next.setDate(row.repeat_day);
    next.setHours(H, M, 0, 0);
    break;
  }
}
```

`daysUntil` の `|| 7` がポイントだ。`(targetDay - curDay + 7) % 7` は「今日が対象曜日と一致」した場合に `0` になる。発火済みの回の「次」を計算しているので、0のままだと同じ日にもう一度発火してしまう。`|| 7` で「最低でも1週間先」に丸めている。

### 2. onceタイプは`is_done`で使い捨てにする

```javascript
if (row.repeat_type === 'once') {
  db.prepare(`
    UPDATE scheduler_schedules
    SET is_done=1, last_run_at=?, last_result=?, updated_at=CURRENT_TIMESTAMP
    WHERE id=?
  `).run(nowStr, lastResult, row.id);
}
```

このシリーズの副業パイプライン自体も、最初の棚卸しタスクは `once` タイプ（`id=36`）で登録されていた。一度実行したらそれきりで、テーブルに「実行済みの記録」だけが残る。

### 3. ログはエージェントごとに個別ファイル

```javascript
const agentLogPath = path.join(LOG_DIR, `sched_${row.id}_${row.dept_id}_${nowStr.replace(/[T:]/g, '-')}.log`);
```

`output/system/sched_16_fukugyo_publisher_2026-07-06-06-01.log` のような命名で、どのIDのどのエージェントがいつ動いたログかが後から一目でわかる。19エージェントが同時に深夜稼働しても、ログが混ざらない。

## この章のまとめ

| 課題 | 解決策 |
|------|--------|
| スケジュールが複雑で読めない | DB一元管理・cronは5分ごとに見に行くだけ |
| 二重起動 | ロックファイル（4分＝cron間隔5分より短く設定） |
| ヘッドレス起動が固まる | `-p` + `IS_SANDBOX=1` を必須化 |
| 繰り返しスケジュールの次回計算ミス | `daysUntil || 7` で「発火済みの当日」を除外 |

「AIが深夜に勝手に働く」ように見えるこの仕組みの正体は、5分おきに起動する162行のNode.jsスクリプトと、SQLiteの1テーブルだ。魔法ではなく、地味なロック処理と時刻計算の積み重ねでできている。

## 次の章へ

第5章では、実際に投稿まで担当している`fukugyo_publisher`エージェントのPlaywright自動化——note.comやXへのログインセッションの持たせ方、ボタンクリックが謎のオーバーレイにブロックされて失敗した実話——を解説する。
