---
title: "Claude DesktopのCoworkでブラウザ操作がPermission deniedになる原因と解決策"
emoji: "🔐"
type: "tech"
topics: ["claude", "anthropic", "chrome拡張", "AIエージェント", "デバッグ"]
published: true
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

## 補足：preflightスキルと今回の問題発見

### preflightスキルとは

筆者のチームでは、長時間の自動タスク（E2Eテスト全カテゴリ実行等）を開始する前に、必要な権限を一括取得する**preflightスキル**を運用しています。

Coworkでは、MCP（Notion・Slack）やブラウザ操作、ローカルファイルアクセスなど、多くのツールで初回に許可プロンプトが表示されます。長時間タスクの実行中にプロンプトが出ると、ユーザーが気づくまでタスクが停滞してしまいます。preflightの目的は、**ダミー操作で事前に許可プロンプトを発火させ、本作業中の停滞を防ぐ**ことです。

具体的には、各ツールに対してread-onlyまたは副作用のない最小限の操作を1回ずつ実行します。

- **MCP接続** — Notion検索・取得・更新（no-op）、Slack検索
- **ブラウザ操作** — 対象ドメインへのナビゲーション（開発環境の各サービス、Notion、GitHub）
- **ローカルファイル** — 設定ファイル・リポジトリの存在確認
- **コマンド実行** — git status等

実行結果はカテゴリ別に集計されます。

```
合計: 13 / 13 成功
→ Preflight完了 — セッション準備OK
```

### 副産物としての異常検知

preflightは権限の一括取得が本来の目的ですが、今回はバージョンアップ後の実行で**ブラウザ操作が6項目すべて失敗**するという結果になりました。

```
• MCP（Notion/Slack）: ✅ 4/4 成功
• ブラウザ: ❌ 0/6（Chrome拡張の問題）
• ローカルファイル: ✅ 2/2 成功
• コマンド実行: ✅ 1/1 成功
```

「ブラウザだけが全滅」という結果から、問題がChrome拡張の権限周りに限定されていることがすぐにわかりました。もしpreflightなしにE2Eテストを開始していたら、テスト途中でPermission deniedが散発し、原因の切り分けに余計な時間がかかっていたと思います。

権限の一括取得を目的としたスキルが、結果として**環境のヘルスチェック**としても機能した形です。

### バージョンアップ時のリセットに注意

Claude Desktopのバージョンアップ時に、`claude.ai/settings/browser-extension` の設定がリセットされることがあります。今回の問題もこれがトリガーでした。

アップデート後にCoworkのブラウザ操作が突然動かなくなった場合は、まずこの設定を確認してください。preflightのような事前チェックを長時間タスクの前に実行する運用にしておくと、こうしたリグレッションに気づきやすくなります。

## おわりに

この問題は、**設定ファイルを見ただけでは原因がわからない**厄介なケースでした。Chrome拡張の難読化されたJSを読み解いて初めて、サイドパネルとCoworkで異なる権限フローが使われていること、サーバー側のドメインカテゴリによるフィルタリングが存在することが判明しました。

同様の症状（サイドパネルでは動くのにCoworkで動かない）に遭遇した方の参考になれば幸いです。
