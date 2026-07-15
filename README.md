# claude-plugins

個人用の Claude Code プラグイン・マーケットプレイス。全 PC から自作スキルを参照するための器。

## 収録プラグイン

- `analysis-skills` — 分析スキル群
  - `moshimo`：プロダクト/データの仮説を実測された結論に落とす調査規律（呼び出し名 `/moshimo`）
  - `deletion-check`：削除差分の消し忘れ（孤児シンボル/連鎖デッドコード）を複数レンズで検出する規律（呼び出し名 `/deletion-check`）
  - `explain-like-im-5`：未知領域の概念を前提ゼロから段階的に腹落ちさせる。深さは会話履歴から相手の level を推定して校正する（呼び出し名 `/explain-like-im-5`）

## 各マシンでの導入

```bash
/plugin marketplace add TrueRyoB/claude-plugins
/plugin install analysis-skills@trueryob   # User スコープ＝全プロジェクトで有効
```

## 新マシンで自動有効化（任意）

`~/.claude/settings.json` に追記（settings.json を dotfiles で同期する運用と相性が良い）：

```json
{
  "extraKnownMarketplaces": {
    "trueryob": {
      "source": { "source": "github", "repo": "TrueRyoB/claude-plugins" }
    }
  },
  "enabledPlugins": { "analysis-skills@trueryob": true }
}
```

## スキルの追加/更新

`plugins/analysis-skills/skills/<name>/SKILL.md` を足す or 直す → commit/push。
各マシンで `/plugin marketplace update` すると反映される。
バージョンを上げたい場合は `plugins/analysis-skills/.claude-plugin/plugin.json` の `version` を更新。
