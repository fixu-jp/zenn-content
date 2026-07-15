---
title: "集計バッチが本番の書き込みを止めていた — INSERT...SELECT と分離レベルの意外な関係"
emoji: "🔒"
type: "tech"
topics: ["mysql", "aurora", "innodb", "performance", "laravel"]
published: false
---

## はじめに

本番データベースのスロークエリを計測していて、面白い波形を見つけました。15分毎に走る集計バッチの完了と**同一秒**に、ユーザー操作由来の INSERT / UPDATE が数十秒の待ちから一斉に解放されている——つまり、集計バッチが実行中の間ずっと、本番の書き込みを堰き止めていたのです。

犯人は集計バッチの `INSERT ... SELECT` 文でした。「SELECT して集計した結果を書き込むだけ」の、一見なんの変哲もない文です。しかし MySQL（InnoDB）のデフォルト分離レベルである REPEATABLE READ では、この文は**読み取り元テーブルの行に共有ロックを張ります**。読んでいるだけのつもりのテーブルが、実はロックされている。

先に持ち帰ってほしいことを書いてしまいます。

> **INSERT ... SELECT は「ただの SELECT + INSERT」ではない。デフォルト分離レベルでは読み取り元テーブルに共有ロックを張り、その間の書き込みを止める。ただし、そのロックが存在する理由（statement-based バイナリログの再現性）が自分の環境に当てはまるかを公式ドキュメントで確認すれば、分離レベルの 1 行で安全に消せる。**

今回はこの結論に至るまでの計測・仕様の裏取り・検証方法を、Aurora MySQL の環境を題材に書きます。題材の具体は抽象化しますが、集計バッチと OLTP 書き込みが同じ writer に同居している構成なら、どこでも起こりうる話です。

---

## 1. 観測 — スロークエリログの Lock_time が語る「保持側」と「待ち側」

発端は、ユーザー操作由来の書き込み（売上明細テーブルへの INSERT / UPDATE）が時折数十秒待たされるケースが計測で可視化されたことでした。スロークエリログを時系列で並べると、こういう構図が浮かびます。

```
# 集計バッチ側（保持側）
# Query_time: 49.2  Lock_time: 0.001
INSERT INTO summary_table (...) SELECT ... FROM sales_table ... GROUP BY ...;

# 同一秒に解放された書き込み（待ち側）
# Query_time: 44.2  Lock_time: 44.1
INSERT INTO sales_table (...) VALUES (...);
# Query_time: 33.9  Lock_time: 33.9
UPDATE sales_table SET ... WHERE id = ...;
```

ここで読み方のポイントが `Lock_time` です。

- **保持側**は `Query_time` が大きく `Lock_time` がほぼゼロ。ロックを「待っていない」＝自分が握っている側です。
- **待ち側**は `Query_time ≒ Lock_time`。実行時間のほぼ全てがロック待ちです。

そして待ち側の解放時刻が、保持側の完了時刻と同一秒に揃う。この時刻一致が「集計バッチがロックを握り、書き込みが待たされている」ことの決定的な証拠になりました。バッチは 15 分毎に走り、1 回 47〜99 秒かかっていたので、**常時 5〜10% の時間帯**で書き込みがこのリスクに晒されていた計算になります。

なお、InnoDB のロック待ちタイムアウト（`innodb_lock_wait_timeout`）はデフォルト 50 秒です。「最大 44 秒待ち」という観測値は、タイムアウト直前まで待った波形そのものでした。50 秒を超えたものは `Lock wait timeout exceeded` エラーとして表面化します。

> 学び: スロークエリログは `Query_time` と `Lock_time` の比で「保持側」と「待ち側」を見分けられる。待ち側の解放時刻と保持側の完了時刻の一致が、因果の証拠になる。

---

## 2. 真犯人 — INSERT ... SELECT は読み取り元に共有ロックを張る

「でも集計バッチは SELECT して INSERT しているだけで、読み取り元テーブルには書いていないのに、なぜロックが？」——ここが今回の核心です。

MySQL 8.0 のリファレンスマニュアル「[Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)」に、はっきり書いてあります。

> INSERT INTO T SELECT ... FROM S WHERE ... (中略) If the transaction isolation level is READ COMMITTED, InnoDB does the search on S as a consistent read (no locks). **Otherwise, InnoDB sets shared next-key locks on rows from S.**

つまり、分離レベルが READ COMMITTED **以外**（＝デフォルトの REPEATABLE READ を含む）のとき、`INSERT INTO T SELECT ... FROM S` は **S の行に共有ネクストキーロックを張る**のです。共有ロックなので他の読み取りは通しますが、同じ行への INSERT / UPDATE / DELETE は排他ロックが取れずに待たされます。

今回のバッチは全件を GROUP BY で舐める集計だったので、実質的に読み取り元テーブルの広い範囲がロック対象でした。集計スキャンに 47〜99 秒かかる＝その間ずっと共有ロックが保持される、という構図です（単文の autocommit なので、ロックは文の完了時に一括解放されます。これも「同一秒に一斉解放」という波形と一致します）。

`CREATE TABLE ... SELECT` や `REPLACE INTO ... SELECT` も同族で、同じ節に同様の記載があります。「SELECT を含む書き込み文」全般が持つ性質だと理解しておくのが安全です。

> 学び: INSERT ... SELECT は REPEATABLE READ では読み取り元の行に共有ネクストキーロックを張る。「読んでいるだけ」に見えるテーブルが、書き込みをブロックする。

---

## 3. なぜそんなロックが必要なのか — 理由は「statement-based バイナリログ」

このロック、嫌がらせではなく明確な存在理由があります。同じマニュアルの続きにこうあります。

> InnoDB has to set locks in the latter case: In roll-forward recovery from a backup using a statement-based binary log, every SQL statement must be executed in exactly the same way it was done originally.

**statement-based バイナリログ**（SQL 文そのものを記録するレプリケーション / リカバリ方式）では、リカバリ時に各 SQL 文が元の実行と完全に同じ結果を再現しなければなりません。もし `INSERT ... SELECT` の実行中に読み取り元テーブルが並行更新されると、リカバリ時の再実行で異なる結果が生まれてしまう。だから読み取り元を共有ロックで凍結して「文の再現性」を守っているわけです。

逆に言えば——**この理由が成立しない環境では、ロックは不要**です。実際、「[Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)」にはこうあります。

> To perform a nonlocking read, set the isolation level of the transaction to READ UNCOMMITTED or READ COMMITTED to avoid setting locks on rows read from the selected table.

READ COMMITTED にすれば、`INSERT ... SELECT` の読み取り部は**ロックを張らない consistent read**（文の開始時点のスナップショット読み）になります。集計としての一貫性は文単位のスナップショットで保たれたまま、ロックだけが消える。

> 学び: このロックの存在理由は statement-based バイナリログの再現性。理由が分かれば「理由が成立しない環境なら外せる」という判断ができる。

---

## 4. Aurora ではロックの根拠が本質的に消えている

ここで自分たちの環境（Aurora MySQL）を見直します。

Aurora のクラスタ内レプリケーションは、バイナリログではなく**ストレージ層**で行われます。リーダーインスタンスはライターと同じ分散ストレージのページを読むため、クラスタ内の複製にバイナリログは使われません。バイナリログが必要になるのは外部レプリケーションなど限られた用途で、今回のクラスタではパラメータグループ上バイナリログは無効でした。

つまり整理するとこうなります。

| 前提 | 従来の MySQL（statement-based binlog） | 今回の Aurora（binlog 無効） |
|---|---|---|
| ロックの根拠（文の再現性） | 成立する | **本質的に存在しない** |
| REPEATABLE READ での INSERT...SELECT | 共有ロック必要 | 共有ロックだけが残っている |
| READ COMMITTED 化 | binlog が ROW 形式なら可 | 制約なく可 |

「かつて必要だった安全装置が、アーキテクチャの変化で根拠を失ったまま、デフォルト設定として残り続けて書き込みを止めている」——という構図です。なお自環境で確認する場合は、`log_bin` の有効/無効と `binlog_format` を見てください。binlog が有効で STATEMENT 形式の場合は、この後の対処（READ COMMITTED 化）を安易に適用してはいけません（そもそも MySQL 側が unsafe な組み合わせとして拒否します）。

> 学び: ロックやガードの「存在理由」がアーキテクチャ変化後も成立しているかを問い直す。Aurora のクラスタ内複製は binlog 非依存で、statement 再現性という根拠は本質的に消えている。

---

## 5. 解決は 1 行 — バッチセッションだけ READ COMMITTED にする

対処は拍子抜けするほど小さくなります。集計バッチの接続に対して、実行前に 1 文流すだけです。

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

ポイントは影響範囲の閉じ方です。

- **`SET SESSION` なのでこの接続だけに効く。** グローバル設定やアプリ全体の分離レベルは変えません。REPEATABLE READ を前提に動いている他の処理には一切波及しない。バッチは短命の CLI プロセスなので、セッションはバッチ終了とともに消えます。
- **集計の一貫性は維持される。** READ COMMITTED でも consistent read は「文の開始時点のスナップショット」を読むため、単文の集計が中途半端な状態を読むことはありません。集計→書き込みが 1 文（今回の場合は `INSERT ... SELECT ... ON DUPLICATE KEY UPDATE` の UPSERT）で完結しているなら、原子性もそのまま。
- **ロールバックは 1 行の revert。** 挙動は REPEATABLE READ 時代に戻るだけで、データ不整合の心配がありません。

Laravel から使う場合はひとつだけ注意があります。read/write split（`read` / `write` の接続分離）を使っている場合、`DB::select()` は read 用 PDO に、`DB::statement()` は write 用 PDO に流れます。分離レベルの設定は**本体の INSERT ... SELECT と同じ PDO に流れる必要がある**ので、`DB::statement()` 同士で揃えます。

```php
// 同じ write PDO に流れるよう、どちらも statement() で発行する
DB::statement('SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED');
DB::statement($insertSelectSql);  // 集計 UPSERT 本体
```

`SET SESSION` を `DB::select()` で発行してしまうと read 用 PDO のセッションだけが変わり、write 側で実行される本体には効かない——という空振りが起こりえます。

> 学び: 対処はバッチセッション限定の分離レベル変更 1 行。ただし read/write split 環境では「同じ PDO に流れているか」まで確認する。

---

## 6. 検証 — 負荷ツールではなく、2 本の並行トランザクションで決定論的に

この修正の効果検証、最初に思いつくのは負荷ツール（ab など）で高並行アクセスを流して before/after を比べる方法ですが、以前別件で「検証環境の vCPU が小さいと CPU 飽和が支配的になり、ロックの効果が計測に現れない」という学びを得ていました。対照ページも同じように遅くなるので、差が見えないのです。

そこで今回は、**2 本の並行トランザクションでロック待ちを決定論的に再現する**方法を取りました。データ量にも環境スペックにも依存しません。

**セッション A（バッチ役・修正前の再現）:**

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- 既定値を明示
START TRANSACTION;
INSERT INTO summary_table (...) SELECT ... FROM sales_table ... ON DUPLICATE KEY UPDATE ...;
-- ここで COMMIT せずに保持（共有ロックを握り続ける）
```

明示トランザクションで COMMIT しないことで、ロック保持時間を人工的に延ばします。本番ではスキャンに 47〜99 秒かかるから待ちが観測できるわけですが、検証環境はデータが小さく一瞬で終わってしまう。COMMIT を保留すればデータ量に関係なくロック窓を作れます。

**セッション B（ユーザー書き込み役）:**

```sql
INSERT INTO sales_table (...) VALUES (...);   -- ブロックされるはず
```

**観測（第 3 のセッション）:**

```sql
SELECT * FROM performance_schema.data_lock_waits\G
SELECT * FROM sys.innodb_lock_waits\G   -- blocking / waiting の対応が一目で分かる
```

期待どおりなら、B が LOCK WAIT で停止し、`sys.innodb_lock_waits` の blocking 側に A の INSERT ... SELECT が現れます（待ち時間を短くしたければ `SET SESSION innodb_lock_wait_timeout = 5;` で観測を高速化できます）。これが Red（現状の問題挙動の再現）。

次にセッション A を `READ COMMITTED` に変えて同じ手順を踏むと、今度は **B が待たされずに即完了**します。これが Green。修正の効果が、負荷や偶然に依存しない形で白黒つきます。

> 学び: ロックの検証は負荷ツールより並行トランザクション 2 本が確実。`performance_schema.data_lock_waits` / `sys.innodb_lock_waits` で「誰が誰を待たせているか」を直接観測する。

---

## まとめ — チェックリスト

最後に、同じ構図を踏まないためのチェックリストを置いておきます。

- [ ] **集計バッチと OLTP 書き込みが同じ writer に同居していないか。** 同居しているなら、バッチの SQL に `INSERT ... SELECT` / `CREATE TABLE ... SELECT` / `REPLACE ... SELECT` が含まれるかを確認する。
- [ ] **含まれるなら、実行時の分離レベルを確認する。** REPEATABLE READ（デフォルト）なら読み取り元テーブルに共有ロックを張っている。スロークエリログの `Lock_time` で保持側/待ち側の実態を計測する。
- [ ] **binlog の状態を確認する。** `log_bin` 無効、または ROW 形式なら、バッチセッション限定の READ COMMITTED 化でロックを外せる。STATEMENT 形式なら別のアプローチ（読み取りをリードレプリカに逃がす等）を検討する。
- [ ] **集計の一貫性要件を確認する。** 文単位のスナップショットで足りるか（定期リフレッシュの表示用集計ならほぼ足りる）。書き込みが UPSERT で冪等なら、再実行も安全。
- [ ] **効果検証は並行トランザクションで決定論的に。** 負荷ツールは環境スペックに結果が左右される。

今回いちばん残ったのは、技術的なテクニックよりもこの感覚です。

> **デフォルト設定の裏には理由がある。そして理由には賞味期限がある。「なぜこのロックが存在するのか」まで掘って、その理由が自分のアーキテクチャで今も成立しているかを公式ドキュメントで確認すれば、大手術に見えた問題が 1 行で安全に解けることがある。**

数十秒のロック待ちを前に「バッチを夜間に逃がすか」「テーブルを分けるか」といった構造変更から考え始めていたら、ずっと大きな工事になっていたはずです。仕様の一次情報に当たる 30 分が、いちばん効率のいい投資でした。

---

## 関連記事

- [SELECT同名列とPDOの後勝ち — 「DBの生の値」を信じて根本原因を読み違えた話](https://zenn.dev/fixu/articles/pdo-fetch-assoc-duplicate-column-last-wins) — 同じく「一見無害な SQL の仕様」に足を掬われた実録。思い込みではなく実機と仕様で確定させる進め方も共通です。
- [「推測」から「計測」へ ― AIネイティブ時代における意思決定スタンスのアンラーニング](https://zenn.dev/fixu/articles/measure-dont-guess-ai-native) — 本記事の出発点になった「まず計測で可視化する」というスタンスの話。
