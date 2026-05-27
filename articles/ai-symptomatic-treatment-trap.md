---
title: "AIは対症療法に陥りやすい — 7連続fixが治らなかった理由と素朴な質問の価値"
emoji: "🩹"
type: "tech"
topics: ["claude", "AIエージェント", "リファクタリング", "アーキテクチャ", "振り返り"]
published: false
---

## はじめに

少し前に [LLM agent に誤前提が 17 連鎖した話 — typo 1 文字が生む指摘の連鎖と、その断ち方](https://zenn.dev/fixu/articles/ai-design-premise-verify-chain) という記事を書きました。あれは「**上流の typo が下流まで通り抜けて、設計時の誤前提が連鎖した話**」でした。

今回はその姉妹編です。同じ AI agent と同じ運用で、別機能の **実装中の動作確認** を進めていたところ、今度は **症状を順次つぶす局所 fix が 7 連続で積み上がって、それでも根本問題が解消しなかった** という事象に遭遇しました。前作が「設計時の誤前提連鎖」だとすれば、今回は「**実装時の対症療法連鎖**」です。

先に手触りを書いてしまうと、

> **AI は目の前のエラーに対する局所 fix を積み重ねがちで、「そもそもこの check が必要か?」という一段抽象化の問いは、ユーザーの素朴な questioning なしには出てきにくい。**

7 連続の fix を Phase A〜G として時系列で並べたあと、なぜ AI は症状修正に陥りやすいのか、構造的な 5 つの理由と効く counter-measure を書き残します。AI に耳の痛い話ですが、自分自身もユーザーとして 7 連続を黙認した側なので、自虐込みの振り返りでもあります。

---

## 1. 何が起きたか — Phase A〜G、7 連続の対症療法

舞台は LINE 多拠点リッチメニュー機能の dev02 実機 verify でした。「親会社の OA (公式アカウント) のリッチメニューから店舗を選ぶ → 該当する子会社の Mini App に遷移する」という導線を作っていました。

ある日の verify で、LINE app の「店舗を選ぶ」 tile をタップすると、子会社側 Mini App の Top.vue に「**LINE Mini App 経由でアクセスしてください**」というエラーが出る、という症状が報告されました。

ここから AI と私の協働で fix を積み重ねたのですが、結論から書くと **7 phase deploy しても、既存の stuck user は解消しませんでした**。順を追って何を直したかを並べます。

### Phase A: 設計違反 fix

`ParentSpaceController@redirectToChild` から子 OA への LIFF redirect 処理を撤去しました。親 OA から子 OA への遷移は本来 LINE app の URL scheme で完結するべきで、Laravel 側で redirect を握る設計は最初から間違っていました。「設計違反を 1 つ消した」きれいな fix です。

### Phase B: URL build bug fix

console 側で、親 OA リッチメニューの action URL を組み立てるときに **子 mini_app_id を埋めてしまっていた** バグを見つけて、親 mini_app_id を使うように修正しました。これも明らかな bug fix です。

### Phase C: latent bug fix

`LiffState` の `ALLOWED_DISPATCH_PATH_PREFIXES` に `/parent-space` が含まれていなかったため、parent-space への dispatch が弾かれていました。allow list に追加しました。

### Phase D: latent bug fix

`routes/line.php` の internal dispatch (= 同一 process 内で別 route に転送する処理) で `$_COOKIE` を preserve していませんでした。dispatch 先で session_id が引き継がれず、session が空になる挙動です。preserve するように修正しました。

### Phase E: typo bug fix (これが象徴的)

`SignupController` と `AuthController` の中に、こんなコードがありました。

```php
// 「LINE_USER_ID を session から消す」のつもりで書かれていたが…
Session::flash('LINE_USER_ID');
```

書いた人の意図はおそらく「session の `LINE_USER_ID` を削除する」だったと思われます。が、Laravel の `Session::flash($key)` は引数 1 つだけで呼ぶと「`$key` に `true` を書き込み、次の request 終了時に削除する」という挙動になります。**削除どころか書き込み、しかも値は `true` という意味不明な状態** が、次の request まで生きてしまう。

削除のつもりが書き込みになっていた、という古典的な typo bug でした。撤去しました。

### Phase F: UX 改善

`TOKEN_EXPIRED` 時に「LINE Mini App 経由でアクセスしてください」という不親切なエラー文を出していたので、「セッションが切れています」に書き換えました。これは本筋ではないが、user-facing メッセージとして改善。

### Phase G: stale session への band-aid

既存の stuck user (session が古い state で固まっているユーザー) に対する対応として、Auth 経由で `session.LINE_USER_ID` を自動 rehydrate する処理を入れました。

これで「7 phase 全部 deploy したから治っているはずだ」と verify したところ、**既存 stuck user の症状は解消しませんでした**。

### 7 phase が直したのは何だったか

冷静に並べると、A〜G のどれも「目の前のエラーログから辿れる症状」を直しています。

- Phase A: 設計違反 → 撤去
- Phase B: URL の組み立て間違い → 訂正
- Phase C: allow list の漏れ → 追加
- Phase D: cookie preserve 漏れ → 追加
- Phase E: typo bug → 撤去
- Phase F: error message が不親切 → 改善
- Phase G: stale session → band-aid

どれも「正しい fix」ではあるのです。実際、回帰テストの観点ではどれも価値がある変更です。問題は **どれも「症状」を直しているだけで「そもそも何故 LINE_USER_ID に依存している?」という上位の問いが立っていなかった** ことでした。

---

## 2. 素朴な質問が architecture refactor を引き出すまで

7 phase 終わって既存 stuck user が解消しなかった事実を見て、私はようやく一段抽象化した疑問を投げました。

> 店舗を選ぶ画面を開くときには、他の画面 (例: アカウント) と同様に、**ログインさえしていればページを開けることが期待値ではありませんか?** 店舗を選んだ後に、別店舗を利用するためのセッション情報を INSERT する、というのが全体フローなのでは? アーキテクチャ自体を一度整理してください。

この瞬間に、AI も私も「あ」となりました。

整理するとこういうことです。

| 画面 | 期待される認可レイヤー | 実装上の実態 |
|---|---|---|
| アカウント画面 | Auth (ログイン済みかどうか) | Auth に依存 |
| 各種設定画面 | Auth | Auth に依存 |
| **店舗を選ぶ画面** | Auth で十分なはず | **session の `LINE_USER_ID` に依存** |

「店舗を選ぶ」画面だけが、なぜか Auth ではなく `session.LINE_USER_ID` というその場限りの状態に依存して認可を行っていたのです。だから session が一度でも壊れる/期限切れになる/cookie が preserve されないと「LINE Mini App 経由で…」エラーになる。

Phase A〜G は全て「session.LINE_USER_ID が消えないように、各経路で preserve / restore する」というアプローチで band-aid を当てていました。本来やるべきは「**店舗選択画面は他画面と同じく Auth ベースで認可する。LINE_USER_ID は店舗選択 *後* のセッション INSERT 用のパラメータに過ぎない**」というアーキテクチャ整理でした。

ここから Phase H として、Auth-based の根本 refactor に着手することになりました。

振り返れば、Phase B の時点で「あれ、なぜ親 mini_app_id じゃないと開けないんだっけ?」と一段抽象化する余地はありました。Phase D の cookie preserve 漏れに気づいた時点で「そもそも cookie に依存しないと開けない設計が変では?」と問えました。Phase E の `Session::flash` typo を見た時点で「LINE_USER_ID を session 経由で運ぶ設計自体に無理があるのでは?」と疑えました。**どの phase でも、一段上の問いを立てる余地はあったのに、立たなかった。**

---

## 3. なぜ AI は症状修正に陥りやすいのか — 構造的 5 理由

ここから本論です。「AI 慎重に使えば良いだけでは?」で済ませず、なぜ症状修正に陥りやすいかを構造的に書きます。心当たりがあれば各ツール (Claude Code / Cursor / Cline / Aider) でも同じ罠は発動しているはずです。

### 理由 1: feedback loop の即時性 bias

目の前にエラーログがあると、「このログを消す fix」までの path は最も短く、最も自然に書けます。
「そもそもこの check が必要か?」「この値を session で持ち運ぶ設計が妥当か?」のような一段抽象化された問いは、cognitive cost が高い上に、答えが出ても **即時の検証手段がありません** (= 大規模 refactor になるので 1 ターンで verify できない)。

AI の挙動はこの即時性 bias に強く影響されます。「次の verify で動くかどうか」をゴールにすると、ゴールに最短で届く path が選ばれる。それが local fix の積み重ねです。

### 理由 2: 訓練データの偏り

OSS の Pull Request は「bug report → patch → merge」のサイクルが大量に学習データとして存在します。GitHub の PR 1 件 1 件は症状単位の小さな修正であり、「アーキテクチャを整理しなおす」種類の変更は PR ではなく **設計ドキュメント + 大規模 refactor PR (めったに無い)** という形でしかリポジトリに残りません。

つまり統計的に、AI が学習している「fix とはこういうもの」のモデルは、本質的に **症状単位の局所 patch** に偏っています。アーキテクチャを整理する PR は train data 上は稀少種です。これは Anthropic / OpenAI 側でも data curation で補正しようとしているはずですが、自然発生する fix の絶対量に押されます。

### 理由 3: 「skin in the game」がない

人間エンジニアは半年後の自分のために綺麗に書こうとします。負債を残せば自分が刺されるからです。

AI には半年後がありません。current session で task を解決すれば「完了」と認識して終わります。Phase A から G まで積み上がった band-aid が将来どれだけの負債になるか、AI には体感がありません。これは AI を責める話ではなく、構造としてそうなっているという話です。

人間側でこれを補わなければ、AI agent は構造的に「目先の動作」最適化を続けます。

### 理由 4: 確認 bias の amplification

これが最も厄介な性質です。Phase A で「ParentSpaceController の redirect が root cause だ」と一旦認識すると、次の verify で別のエラーが出ても、AI は「あ、別の bug が出てきた」と認識します。**「先程の root cause 認識が間違っていたのでは?」とは認識しません。**

各 phase で「root cause を特定 → fix → 次の bug が出る → 次の root cause を特定」を繰り返しているうちに、「7 つも root cause があった」という奇妙な結論が積み上がります。冷静に考えれば「root cause がそんなにたくさんあるはずがない、共通の上位構造があるのでは?」と self-questioning すべきところを、各 phase の局所成功体験が self-questioning を抑制します。

ユーザーが介入して「**7 個も root cause が見つかること自体が変ですよね?**」と問うまで、自己修正が起こらない。

### 理由 5: 「正しい abstraction」のモデル不在

「`LINE_USER_ID` は Auth とは別レイヤーの concern (= LINE 環境固有の identity 情報) なので、認可とは分離すべき」のような architectural insight は、業務的に「**ユーザーの identity とは何か**」「**認可と認証の責務分離**」を理解していないと出てきません。

AI はパターン認識は強いのですが、**パターン自体が誤っている時にそれを検出する** のは苦手です。なぜなら「正しい abstraction」のモデルがそもそも明示的に与えられていないからです。「authentication と authorization は別レイヤー」と教科書で読んだ知識は持っていても、目の前のコードが「session.LINE_USER_ID で認可している」と気づいて違和感を感じる、というところまでは行きにくい。

これは 1 ターンの design review なら反応するのですが、**7 phase の fix を順次積み上げる流れの中では「違和感センサー」がオフになっている** という点が問題です。

---

## 4. 効く counter-measure

5 理由を整理した上で、現場で効いた / 効きそうな対策を 5 つ書きます。

| 対策 | 何をやるか | 発動条件 | コスト |
|---|---|---|---|
| 素朴な疑問の投げ込み | 「他の画面と同様にログインしてれば開けるのが期待値ですよね?」 | fix が 3 連続で当たらなかった時 | 低 (= 1 発言) |
| 追い打ち質問で逃げ道封じ | 「Auth ベースではなく LINE_USER_ID に依存している理由を教えて」 | AI が「別の root cause がありました」と言い始めた時 | 低 |
| fix 回数の上限 | Phase 3 で一旦止めて「待て、何かおかしい」と聞く | rule として事前合意 | 中 (= 流れを止める cognitive cost) |
| メモリ / skill での protocol 強制 | 設計前提の実機 verify 必須ルール、3 連続 fix 後の self-questioning ルール | 永続化 | 中 (= rule 管理コスト) |
| Pre-mortem の強制 | 「この fix を deploy した後、何が次に壊れる?」を AI に答えさせる | 各 fix の deploy 前 | 中 |

### 素朴な疑問の威力

実は今回のケースで効いたのは、技術的に高度な指摘ではなく **超素朴な「他の画面と同じ作りでよくないですか?」** という疑問でした。

これが効く理由は理由 1〜5 全てに同時に効くからです。即時性 bias を破り (= 即時の検証に直結しない問い)、訓練データのパターン外の問いを投げ (= 「PR を書く」ではなく「設計を疑う」)、半年後の視点を持ち込み、確認 bias を解除し (= 既存の root cause 認識を全否定する)、「正しい abstraction とは何か」の議論に強制的に引き上げます。

専門用語を使わない素朴な問いの方が、これら全てに同時に効きます。

### Pre-mortem は事前と事後の両方に要る

Pre-mortem の概念は、過去記事 [AIは知っているのに使わない — 設計タスクの "task-kickoff" プロトコルで潜在能力を引き出す](https://zenn.dev/fixu/articles/task-kickoff-ai-design-protocol) で「**設計時** の Phase 0-3」として組み込み済みです。

今回得た追加知見は、**実装中の各 fix 直後にも mini pre-mortem が要る** ということでした。「この fix を deploy した後、何が次に壊れる?」「この fix が間違っている可能性はあるか?」「root cause が別にある可能性はあるか?」の 3 問を、各 fix の commit 前に強制すれば、Phase B あたりで「あれ、同じ画面で似た bug が連続している」と AI 自身が気づける可能性があります。

### fix 回数上限 ≒ 連続失敗時の sentinel

「3 連続で fix を打ったのに verify が通らなかったら、一旦止まって architecture を疑う」というルールは、5 理由のうち **理由 4 (確認 bias の amplification)** に直接効きます。各 phase の局所成功で麻痺しがちな self-questioning を、ルールベースで強制的に発動させる仕組みです。

具体的には skill / CLAUDE.md に書き込んでおけば、AI 側からも「ルール上 3 連続失敗のため、一段抽象化した整理を提案します」と発話するようになります。

---

## 5. まとめ — AI 時代に「素朴な質問」の価値は上がっている

7 phase の対症療法を積んだあと、私が投げた質問はこの一文だけでした。

> 「店舗を選ぶ画面を開くときには、他の画面と同様に、ログインさえしていればページを開けることが期待値ではありませんか?」

技術的にはどうということのない問いです。専門知識を要するわけでもなく、「他の画面と一貫していないのは変では?」というだけの問い。それでも、この問いが Phase A〜G の全ての band-aid を一段上から見直す Phase H refactor に直結しました。

ここから引き出したい結論は、

> **AI 時代に「専門知識を持つ人間が AI に何をやらせるか」よりも、「専門知識を持つ人間が AI の判断に何を questioning するか」の方が成果を決めるようになりつつある。**

ということです。AI は実装と症状 fix を高速に量産します。それ自体は素晴らしい能力です。が、「**この fix の積み上げ方は変では?**」「**そもそもこの設計の前提は正しい?**」という一段抽象化された問いは、AI からは構造的に出にくい。

前作 [LLM agent に誤前提が 17 連鎖した話](https://zenn.dev/fixu/articles/ai-design-premise-verify-chain) は「上流の typo が下流まで通り抜けた」設計時の誤前提連鎖でした。本記事は「下流の局所 fix が層を成して根本 refactor を遅らせた」実装時の対症療法連鎖でした。両方とも真因は同じで、**「一段抽象化した questioning が、人間からの介入なしには起こらなかった**」ことに尽きます。

AI agent ツールが進化するほど、人間側の役割は「コードを書く」ことから「**AI の判断軸そのものを揺さぶる素朴な問いを投げ込む**」ことへとシフトしていく気がしています。コードを書ける人より、素朴な質問を投げ込める人の方が、AI 時代には希少資源かもしれません。

自分を含めて、Phase 3 で止まれなかった全エンジニア仲間に、この記事を捧げます。

---

## 関連記事

- [LLM agent に誤前提が 17 連鎖した話 — typo 1 文字が生む指摘の連鎖と、その断ち方](https://zenn.dev/fixu/articles/ai-design-premise-verify-chain) — 本記事の姉妹編。設計時の上流誤前提連鎖の話。
- [AIは知っているのに使わない — 設計タスクの "task-kickoff" プロトコルで潜在能力を引き出す](https://zenn.dev/fixu/articles/task-kickoff-ai-design-protocol) — 設計時の Pre-mortem を Phase 0-3 として組み込む話。今回得た「実装中の mini pre-mortem」はその追補。
- [AI 多層防御を通り抜けた重大インシデント未遂 — verify 経路の真正性と人間の批判的圧力](https://zenn.dev/fixu/articles/ai-multi-agent-verify-authenticity) — 「人間の批判的圧力」の重要性を verify 経路の文脈で書いた話。本記事の結論と同じ方向。
- [許可リストか、拒否リストか — AI 時代に再発見した『コード変更量最小化』の原則](https://zenn.dev/fixu/articles/allow-list-vs-deny-list-code-minimization) — 「AI は綺麗そうな新規実装を量産しがち」という同根の罠を、許可/拒否リスト選択の文脈で書いた話。
