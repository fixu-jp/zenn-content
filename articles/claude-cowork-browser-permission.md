---
title: "Claude DesktopのCoworkでブラウザ操作がPermission deniedになる原因と解決策"
emoji: "🔐"
type: "tech"
topics: ["claude", "anthropic", "chrome拡張", "AIエージェント", "デバッグ"]
published: false
---

## はじめに

Claude Desktopの**Cowork（ローカルエージェントモード）**からChrome拡張（Claude in Chrome）経由でブラウザ操作を行った際、特定のドメインで以下のエラーが発生しました。

```
Permission denied for JavaScript execution on this domain
Navigation to this domain is not allowed
```

同じドメインに対して**サイドパネルから操作すると正常に動作する**のに、Cowork経由だと拒否される。この記事では、原因の特定から解決までの過程を共有します。

## 症状

| ドメイン | サイドパネル | Cowork |
|---|---|---|
| `app-a.example.dev` | 読み取り・操作 OK | 読み取り・操作 OK |
| `app-b.example.dev` | 読み取り・操作 OK | 読み取り OK / 操作 **NG** |
| `app-c.example.dev` | 読み取り・操作 OK | 読み取り OK / 操作 **NG** |

ポイントは以下の通りです。

- **ページの読み込み自体はできる**（アクセスはできている）
- **JS実行・クリック・スクリーンショット等の操作だけが拒否される**
- **サイドパネルでは同じドメインで全操作が成功する**

## 調査：設定ファイルに制限はなかった

まず疑ったのは、Claude Code側やChrome拡張の設定ファイルでドメインが制限されている可能性です。以下をすべて確認しましたが、制限は見つかりませんでした。

- **Claude Code `settings.json`** — ドメイン制限なし
- **Claude Desktop `claude_desktop_config.json`** — Chrome拡張ペアリング設定のみ
- **Chrome拡張 `manifest.json`** — `host_permissions: <all_urls>` で全URL許可
- **Chrome `Secure Preferences`** — `withheld_permissions: {}` で保留権限なし
- **Chrome managed policies** — 拡張のドメイン制限ポリシーなし
- **Chrome拡張オプションページの「承認済みのサイト」** — 空（0件）

つまり、**静的な設定レベルではどこにもドメイン制限がかかっていない**状態でした。

## 原因：Chrome拡張のランタイム権限フロー

Chrome拡張の難読化されたJavaScriptコード（`mcpPermissions-qqAoJjJ8.js`、`PermissionManager-9s959502.js`）を分析した結果、以下の仕組みが判明しました。

### サイドパネルの場合

サイドパネルから操作する場合、各ドメインへの初回アクセス時に**ユーザーに権限プロンプトが表示**されます。承認すれば「承認済みのサイト」に追加され、以降は自由に操作できます。

### Cowork（ローカルエージェントモード）の場合

Coworkではブラウザ操作がWebSocketブリッジ経由で行われ、**異なる権限フロー**が使われます。

```
Claude Desktop
  → WebSocketブリッジ (bridge.claudeusercontent.com)
    → Chrome拡張 (Service Worker)
      → ブラウザ操作
```

ブリッジ経由のツール呼び出しには、以下のパラメータが含まれます。

```javascript
{
  tool: "navigate",
  args: { url: "https://app-b.example.dev/..." },
  permission_mode: "follow_a_plan",  // ← ポイント
  allowed_domains: ["app-a.example.dev"],  // ← フィルタ済み
  // ...
}
```

### `follow_a_plan` モードの動作

`permission_mode` が `follow_a_plan` の場合、Chrome拡張の `PermissionManager` は以下の処理を行います。

```javascript
// turnApprovedDomains に1つでもドメインがある場合、
// リストにないドメインは即座に拒否（プロンプトも表示しない）
if (turnApprovedDomains.size > 0 && !isTurnApprovedDomain(domain)) {
  return { allowed: false, needsPrompt: false };
}
```

つまり、`allowed_domains` リストに含まれないドメインは**プロンプトすら表示されず即座に拒否**されます。

### `allowed_domains` の生成ロジック

`allowed_domains` リストは、Anthropicのサーバー側APIでフィルタリングされて生成されます。

```javascript
async function filterDomains(domains) {
  const approved = [], filtered = [];
  for (const domain of domains) {
    const category = await fetchCategoryFromAPI(domain);
    // category1, category2, category_org_blocked は除外
    if (!category || (category !== "category1" && category !== "category2" 
        && category !== "category_org_blocked")) {
      approved.push(domain);
    } else {
      filtered.push(domain);
    }
  }
  return { approved, filtered };
}
```

各ドメインについて `https://api.anthropic.com/api/web/domain_info/browser_extension?domain=...` にカテゴリを問い合わせ、**制限カテゴリに分類されたドメインはサイレントに除外**されます。

### まとめると

```
サイドパネル:
  ドメインアクセス → 権限プロンプト表示 → ユーザー承認 → 操作OK

Cowork:
  ドメインアクセス → APIでカテゴリチェック → 制限カテゴリ → allowed_domainsから除外
  → turnApprovedDomainsに含まれない → プロンプトなしで即拒否
```

## 解決策

`claude.ai/settings/browser-extension` に設定画面があります。

1. **「すべてのサイトでデフォルト」** の設定を **「拡張機能を許可」** に変更
2. 必要に応じて、操作時に表示される権限プロンプトで個別ドメインを承認

この設定を変更することで、Coworkからのドメインカテゴリ制限が解除され、サイドパネルと同様に個別の権限プロンプトが表示されるようになります。

:::message alert
**注意:** 「すべてのブラウザアクションを許可」を選択すると全サイトへのアクセスがプロンプトなしで許可されます。セキュリティリスクを考慮し、業務上必要なドメインのみを個別承認する運用を推奨します。
:::

## 補足：preflightスキルによる早期検知

### バージョンアップで設定がリセットされる問題

Claude Desktopのバージョンアップ時に、`claude.ai/settings/browser-extension` の設定がリセットされることがあります。今回の問題もバージョンアップがトリガーでした。

厄介なのは、**設定がリセットされたこと自体に気づけない**点です。Coworkを使おうとして初めてPermission deniedが出て、そこから調査が始まることになります。

### preflightスキルとは

この問題を早期検知するために、筆者のチームでは**preflightスキル**を運用しています。これは、AIエージェントが長時間の自動タスク（E2Eテスト等）を開始する前に、必要な権限がすべて揃っているかを事前チェックする仕組みです。

チェック項目は以下のようなカテゴリに分かれています。

- **MCP接続**（Notion読み書き、Slack投稿）
- **ブラウザ操作**（各ドメインへのナビゲーション、ページ読み取り、クリック、JS実行、スクリーンショット）
- **ローカルファイル**（設定ファイル、リポジトリの存在確認）
- **コマンド実行**（git、SSH等）

実行すると、各項目の成否がまとめて報告されます。

```
現時点の結果まとめ：
• MCP（Notion/Slack）: ✅ 4/4 成功
• ブラウザ: ❌ 0/6（Chrome拡張の問題）
• ローカルファイル: ✅ 2/2 成功
• コマンド実行: ✅ 1/1 成功
```

### 今回の活躍

今回、Claude Desktopのアップデート後にpreflightを実行したところ、**ブラウザ操作が6項目すべて失敗**していることが即座に判明しました。

もしpreflightなしでE2Eテストを開始していたら、テスト途中でPermission deniedが散発し、原因の切り分けに余計な時間がかかっていたでしょう。preflightが問題を**「ブラウザ権限の全滅」**と明確に切り出してくれたことで、調査のスコープを絞ることができました。

### preflightの設計ポイント

preflightスキルを設計する際のポイントをいくつか挙げます。

- **ダミー操作で権限を発火させる** — 実際のタスクではなく、副作用のない最小限の操作（ページ読み取り、テスト用チャンネルへの投稿等）で権限プロンプトを出す
- **カテゴリ別に成否を集計する** — 「どの系統が壊れているか」が一目でわかるようにする
- **長時間タスクの前に必ず実行する** — CIのヘルスチェックと同じ発想で、本作業の前に環境の正常性を確認する

AIエージェントが複数の外部ツール（ブラウザ、MCP、SSH等）に依存する場合、どれか一つの権限が欠けただけでタスク全体が失敗します。preflightはその**単一障害点を事前に洗い出す**ための仕組みです。

## おわりに

この問題は、**設定ファイルを見ただけでは原因がわからない**厄介なケースでした。Chrome拡張の難読化されたJSを読み解いて初めて、サイドパネルとCoworkで異なる権限フローが使われていること、サーバー側のドメインカテゴリによるフィルタリングが存在することが判明しました。

一方で、preflightスキルのおかげで問題の発見と切り分けはスムーズでした。AIエージェントの運用では、こうした**環境の健全性を定期チェックする仕組み**がますます重要になると感じています。

同様の症状（サイドパネルでは動くのにCoworkで動かない）に遭遇した方の参考になれば幸いです。
