---
name: skill-update
description: ローカル(~/.claude/skills)で編集/新規作成した自作スキルを個人marketplace repo(TrueRyoB/claude-plugins)に反映する事務手順。同期・改名・version bump・main直pushまで一括。「skill-update」「スキルを反映/同期」「skill を push」「plugin 更新して」等で発火。
---

# skill-update

ローカル編集した skill を個人 marketplace に反映する。**事務的に・トークン節約・差分と結果だけ報告。posture や思考の narration をしない。** 確認は「改名の旧名が曖昧」な時だけ 1 問。

## 定数（環境が変わったらここだけ直す）
- `REPO=/Users/ryoji.araki/claude-plugins`
- repo skill: `$REPO/plugins/analysis-skills/skills/<name>/SKILL.md`
- plugin.json: `$REPO/plugins/analysis-skills/.claude-plugin/plugin.json`（`version`）
- 名称参照: `$REPO/.claude-plugin/marketplace.json`（description）, `$REPO/README.md`
- ローカル: `~/.claude/skills/<name>/`
- push 先: **main 直**（`git push origin main`）
- 各マシンでの反映: `/plugin marketplace update`

## 引数
`<name>` = 反映するローカル skill 名。省略時は手順1で差分検出して対象を洗い出す。

## 手順
1. 差分検出（1コマンド。CHANGED/NEW が対象）:
```
REPO=/Users/ryoji.araki/claude-plugins; for d in ~/.claude/skills/*/; do n=$(basename "$d"); r="$REPO/plugins/analysis-skills/skills/$n/SKILL.md"; { [ -f "$r" ] && diff -q "$d/SKILL.md" "$r" >/dev/null 2>&1; } || echo "CHANGED/NEW: $n"; done
```
2. 改名判定: repo に旧名 dir があり local に無い ＆ 内容が新名とほぼ一致 → 改名。`cd $REPO && git mv plugins/analysis-skills/skills/<old> plugins/analysis-skills/skills/<new>`。判定が曖昧な時だけ旧名を 1 問確認。
3. 同期: 単一ファイルは `cp ~/.claude/skills/<name>/SKILL.md $REPO/plugins/analysis-skills/skills/<name>/SKILL.md`。サブファイルありは `rsync -a --delete ~/.claude/skills/<name>/ $REPO/plugins/analysis-skills/skills/<name>/`。新規は dir 作成後に同期。
4. 改名時のみ: marketplace.json の description と README.md の旧名 → 新名に更新。
5. version bump: `plugin.json` の `version` を上げる（改名/規律追加=minor、微修正=patch）。
6. commit & push（main 直）:
```
cd $REPO && git add -A && git status -s
git commit -q -m "<type>(analysis-skills): <差分要約>" -m "Co-Authored-By: <このセッションのモデル> <noreply@anthropic.com>"
b=$(git branch --show-current); [ "$b" != main ] && { git checkout main && git merge --ff-only "$b"; }
git push origin main
```
7. 報告（これだけ）: `git diff --stat` 相当の変更ファイル、push の `old..new`、そして「各マシンで `/plugin marketplace update`」の一行。

## やらないこと
- PoC/scratch・repo 外の作業物は反映しない。
- 冗長な説明・思考過程の出力をしない。差分と結果のみ。
