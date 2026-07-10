---
title: "Claude Code を cron で自律稼働させる方法 — DBスケジュールで19エージェントを深夜に自動起動する仕組みをコードで解説"
emoji: "⏰"
type: "tech"
topics: ["claudecode", "ai", "automation", "cron", "nodejs"]
published: true
---

## 5分ごとに1本のNode.jsスクリプトが「今、誰を起こすか」を決めている

このAI営業会社には人間のオペレーターがいない。だが毎朝07:00にX投稿が飛び、毎週月曜09:00に週次レポートが上がり、深夜3時に副業コンテンツが生成されてGitHubにpushされる。これを実現しているのは複雑な仕組みではなく、`crontab`に登録された1行と、それが5分おきに起動する120行ほどのディスパッチャだけだ。

```bash
$ crontab -l | grep dispatcher
*/5 * * * * /usr/bin/node /root/ai-sales-company/scripts/agent_dispatcher.js \
  >> /root/ai-sales-company/output/system/scheduler_dispatcher.log 2>&1
```

`agent_dispatcher.js`（`/root/ai-sales-company/scripts/agent_dispatcher.js`）の中身をそのまま引きながら、「Claude Codeを人間なしで自律稼働させる」実装を解説する。

---

## スケジュールはcrontabではなくSQLiteに置く

19体のエージェントそれぞれに個別のcron行を書く方式は最初から採用しなかった。実行時刻の変更・一時停止・実行内容の編集がすべてcrontab編集＝サーバーへのSSHになるからだ。代わりに、社内ダッシュボード（`office-app`）が管理する`office.db`の`scheduler_schedules`テーブルに1件ずつ登録し、cronは「5分おきにこのテーブルを見て期限が来たものを起動するだけ」の薄いディスパッチャに徹させている。

```sql
CREATE TABLE IF NOT EXISTS scheduler_schedules (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  dept_id TEXT NOT NULL,           -- どのエージェントを起動するか（例: fukugyo_publisher）
  dept_label TEXT NOT NULL,
  instruction TEXT NOT NULL,        -- 起動時に渡す指示文
  scheduled_at TEXT NOT NULL,       -- "YYYY-MM-DDTHH:MM"（JST）
  repeat_type TEXT NOT NULL DEFAULT 'once',  -- once / daily / weekly / monthly
  repeat_day INTEGER,
  is_done INTEGER NOT NULL DEFAULT 0,
  is_enabled INTEGER NOT NULL DEFAULT 1,
  last_run_at TEXT,
  last_result TEXT
);
```

cronはこのテーブルに対して`SELECT * FROM scheduler_schedules WHERE is_enabled=1 AND is_done=0 AND scheduled_at <= ?`を投げるだけで、次に何を起動すべきかを判断する。スケジュールの追加・変更はダッシュボードのフォームから行い、サーバー側の設定ファイルは一切触らない。

---

## 二重起動を防ぐのは「ロックファイルの年齢」

5分間隔のcronと数分かかることもあるエージェントの実行時間が重なると、同じディスパッチャが二重に起動しかねない。対策はロックファイル1つだけだが、単純な存在チェックにはしていない。

```javascript
const LOCK_PATH = '/tmp/agent_dispatcher.lock';
if (fs.existsSync(LOCK_PATH)) {
  const lockAge = Date.now() - fs.statSync(LOCK_PATH).mtimeMs;
  if (lockAge < 4 * 60 * 1000) { // 4分以内のロックは有効（cron間隔5分以内）
    log(`already running (lock age ${Math.round(lockAge/1000)}s), skip`);
    process.exit(0);
  }
}
fs.writeFileSync(LOCK_PATH, String(process.pid));
process.on('exit', () => { try { fs.unlinkSync(LOCK_PATH); } catch (_) {} });
```

ロックファイルが存在するだけで弾くと、ディスパッチャ自体がクラッシュしてロック解除処理（`process.on('exit', ...)`）が走らなかった場合に永久ロックになる。そこで「ロックファイルの更新時刻が4分以内かどうか」を見る。cronの実行間隔（5分）より短い4分をしきい値にすることで、正常系の二重起動は防ぎつつ、異常終了時は次のcronサイクルで自動的に復旧する。

---

## エージェントの起動は`spawn` + `detached: true` + `unref()`

期限が来たスケジュールをどう実行するか。ここが一番実務的に重要な部分だ。

```javascript
const fullInstruction = `深夜自律稼働モード（スケジューラ自動実行 id=${row.id}）: ${row.instruction}`;

const child = spawn(CLAUDE_BIN, ['-p', fullInstruction, '--agent', row.dept_id], {
  cwd: ROOT,
  detached: true,
  stdio: ['ignore', agentLogFd, agentLogFd],
  env: { ...process.env, HOME: '/root', IS_SANDBOX: '1' }
});
child.unref();
```

ポイントは3つ。

1. **`-p`（ヘッドレスモード）は必須。** 対話UIを起動しようとするとcron配下には表示するターミナルがなく起動に失敗する。
2. **`detached: true` + `unref()`。** ディスパッチャ自身はエージェントの完了を待たずに即座に終了する必要がある。cronプロセスの子として紐付いたままだとディスパッチャのプロセス終了時にエージェントも巻き込まれて`kill`される。切り離すことで、エージェントは数十分かかる作業でも独立して走り続けられる。
3. **`env`に`IS_SANDBOX: '1'`を明示的に足す。** root権限で`bypassPermissions`（確認なし実行）を許可するフラグで、これと`-p`の両方が揃わないとcron/spawn配下での起動自体が失敗する。この2つを外し忘れて「ローカルでは動くのにcronからだと動かない」を1度やらかしている。

そして起動指示の先頭に必ず`深夜自律稼働モード（スケジューラ自動実行 id=${row.id}）:`という文字列を差し込んでいる。これは単なる飾りではなく、社内の運用ルール（`CLAUDE.md`）が「この文言があるかどうか」でエージェントの行動範囲を切り替えるための合図になっている。

```
実行OK：Web検索・情報収集・レポート保存・HANDOFF更新・git commit/push
必ず停止：メール/SNS送信・契約・費用発生アクション・既存データ削除・判断困難ケース
例外：fukugyo_publisher / fukugyo_analyst のX投稿・note/Zenn投稿・Slack通知は、
      オーナー承認済みスケジューラ起動タスクに限り実行可
```

通常の対話セッションでは「実行前に必ずオーナーに確認」が大原則だが、cron起動には人間が張り付いていない。かといって全部自動実行を許すと契約や外部送信が暴走するリスクがある。そこで「スケジューラ起動＝深夜モード」という信号1つを渡すだけで、エージェント側のMarkdown定義がその範囲内での自律行動に切り替わる設計にした。承認のたびに人が介在する仕組みと、何でも自動化する仕組みの中間を、プロンプトの先頭1行で実現している。

---

## 繰り返しスケジュールは「次回」を都度計算し直す

`repeat_type`が`once`以外なら、実行後に次回の`scheduled_at`を計算してテーブルを上書きする。cronのように固定パターンを保持するのではなく、毎回「今日から見て次はいつか」を計算し直す方式だ。

```javascript
case 'weekly': {
  const targetDay = row.repeat_day; // 0=Sun..6=Sat
  const curDay = now.getDay();
  const daysUntil = ((targetDay - curDay + 7) % 7) || 7; // 最低1日先
  next.setDate(now.getDate() + daysUntil);
  next.setHours(H, M, 0, 0);
  break;
}
```

`|| 7`が地味に効いている。`daysUntil`が0になるケース（今日がちょうどtargetDayの日）を「今日」ではなく「1週間後」に倒すためのフォールバックだ。これがないと、当日の実行後に同日をもう一度「次回」として登録してしまい、同じ日に複数回発火するバグになる。

---

## この会社のスケジューラ・Hook・全19エージェント設計を解説する本

`agent_dispatcher.js`の全実装・`office-app`のダッシュボードとの連携・スケジュール障害時の切り分け方まで、実際のコードと設定ファイルつきで解説するZennの有料本を書いている。

**[Claude Code で作る AI 自動化会社 完全解説](https://zenn.dev/nakayama_acari/books/ai-company-automation)**（¥4,980）

架空のサンプルではなく、今この瞬間5分おきに動いているVPS上のファイルをそのまま載せている。
