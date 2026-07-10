---
name: deletion-check
description: 機能・コードを削除する差分に対し、複数のレンズ（前方DFS/後方探索/footprint網羅/対称比較 等）を併用して「消し忘れた孤児シンボル（連鎖デッドコード）」と「宙に浮いた壊れた参照」を検出する。tsc/lint/build が緑でも見逃す未使用の union メンバー・export・文字列キー（イベント型 / i18n キー / ルート / GraphQL フィールド / BFFスキーマ）を拾うのが主目的。「削除漏れをチェック」「消し残しがないか」「deletion-check」「この削除で孤児になるものは」等で発火。
---

# deletion-check

削除は**波及する**。あるシンボルの利用箇所を消すと、その定義が孤児（どこからも参照されないデッドコード）になり、その定義が使っていた別のシンボルもまた孤児になる——という**連鎖（cascading dead code）**が起きる。このスキルは削除差分を、複数の独立した**レンズ**で走査して消し忘れを機械的に列挙する。

## なぜ必要か（緑を信用しない）

`tsc` / ESLint / build が緑でも、以下は**原理的に検出されない**ため消し残る:

- **union 型の未使用メンバー**（例: 文字列リテラル union のイベント型）。合法なので型エラーにならない。
- **未使用の export**（`tsc`/ESLint 既定はローカル変数しか見ない）。
- **文字列キー越しの参照**（`eventType: 'X'` / `t('a.b.c')` / ルートキー / GraphQL の operation・fragment・field 名）。AST 解析ツールの死角。

→ 緑は「壊れていない（**必要条件**）」の保証であって「不要物が残っていない（**十分条件**）」の保証ではない。この差を埋めるのがこのスキル。

**そして単一の手法では seed バイアスを消せない。** 起点を「自分が覚えているシンボル」から作ると、その語彙の外は構造的に発見不能になる（下記 L1 だけだと漏れる）。**複数レンズを併用し互いの死角を埋めるのが原則。**

## 思考フレームワーク（削除漏れを見つけるレンズ集）

各レンズは「探索の向き × チェックリストの出所 × ヒットの得方 × 検証」で分類される。状況に応じて複数を重ねる。

### グループ1：参照グラフの探索方向
- **L1 削除起点の前方DFS（孤児検出 / outbound reachability）** — 消したものが参照していた先を辿り、残存参照0になった定義を潰す。→ 詳細手続きは後述の「アルゴリズム」。
- **L2 被参照からの後方探索（dangling reference / dead-link / blast-radius / inbound）** — L1の逆向き。「この機能を**指していた**リンク・ボタン・import・遷移」を列挙し、削除後に宙に浮くもの（デッドリンク・壊れた import）を探す。孤児の**補集合**。移行系の事故源の筆頭。
- **L3 ライブルート起点の全体 mark-and-sweep（tree-shaking / 大域デッドコード）** — 削除差分ではなく、アプリの生きたエントリ（route/entrypoint）から到達可能を全マークし、**未マーク＝死**とみなす。「知らないから seed に入れられなかった」漏れを踏み潰す。

### グループ2：チェックリストの出所（何を網羅の基準にするか）
- **L4 footprint カテゴリ網羅（taxonomy / completeness）** — UI/route/i18n/log/schema/asset/flag/e2e… の軸表から seed を生成。記憶ではなく表から出発する。→ 軸表は後述「手順・footprint 軸表」。
- **L5 生成時 diff の逆再生（inverse of introduction commit / git 考古学）** — 機能を**最初に導入したPR/コミット**を特定し（`git log --diff-filter=A -- <path>` / `git log -S <symbol>`）、「追加された全て」に削除が対応するか突合。削除は生成の逆写像であるべき。創世コミットが最も正確な footprint 台帳。
- **L6 双子・置換先との対称比較（symmetry with replacement）** — 置換・移行の削除では、置換先（例: 旧キーワード画面→新テーマ画面）の要素集合を鏡にして、旧側の対応物が残っていないか照合。
- **L7 実行ライフサイクル・トレース（runtime pipeline walk）** — 機能の一生を実行順に辿る: `routing → 認可/guard → データ取得(BFF/API) → render → ユーザー操作 → イベント発火/ログ → 永続化(cookie/cache)`。時系列なので下流の副作用（ログ・分析・キャッシュ）を取りこぼしにくい。
- **L8 境界越え伝播（cross-boundary / out-of-repo）** — grep で見えない**リポジトリ外**を明示列挙: BFF/バックエンドの resolver・スキーマ、分析基盤(Snowflake)、ログ仕様書(Notion)、feature-flag サービス、CDN アセット、sitemap、監視/アラート、deep link、モバイル parity。

### グループ3：ヒットの得方（探索手段）
- **L9 命名規約の面探索（naming-family lexical sweep）** — 機能が使う命名族（`*Keyword*` / `TAP_KEYWORD_*` / `organization.keyword.*` / slug `invitation/keywords`）を洗い、各族を広く grep して共有資産を差し引く。**文字列キー**に強く AST の死角を埋める。
- **L10 ツール支援の静的到達可能性（knip / ts-prune / graphql-codegen / noUnusedLocals）** — 機械に到達可能性を計算させる。export 取りこぼしに強い（文字列キーは苦手＝L9と相補）。⚠️ 文字列リテラル union メンバー・文字列キー参照は検出できない。

### グループ4：検証（見つけた後・実測）
- **L11 事後不変条件の検証（post-condition invariant）** — 「削除後はこうなっているべき」を宣言して照合: route manifest に痕跡ゼロ / bundle に含まれない / codegen manifest に該当 operation なし。探すのでなく満たすべき状態を検証。
- **L12 テレメトリ突合（telemetry reconciliation）** — 宣言上の footprint と実測を突合: まだ発火中のイベント型は？ まだ叩かれる query は？ PVは0か？「定義はあるが発火実績ゼロ」＝デッド候補。

## モデル（用語を正確に）

- **参照グラフ (reference graph)**: ノード＝シンボル、有向辺 `A → B`＝「A が B を参照する（use → def）」。
- **シンボル**: export / 型・union メンバー / i18n キー / ルートキー / GraphQL field・fragment・operation 名 / testdata 定数 / モジュール（ファイル）。
- **起点集合 (seed set)**: 今回の差分で**削除された**シンボル全部（丸ごと消したファイル＋変更ファイルから消えた行の両方）。L4/L5/L9 で網羅的に導出する。
- **孤児 (orphan)**: そのシンボルへの**残存参照（live referrer）が 0** のもの。＝削除集合を除いた referrer が空。
- **保護集合 (protected set)**: 共有資産など「消してはいけない」シンボル。DFS はここで**枝刈り (prune)** する。PR 説明の「残す共有資産」がこれ。
- 目的＝起点集合から到達する**新規孤児の推移閉包 (transitive closure)** を求めること。JS バンドラの tree-shaking と同じ発想。

## アルゴリズム（L1 の詳細：DFS / ワークリスト法・不動点）

```
removed   = seed set（差分で消えたシンボル）
worklist  = removed のコピー
staged    = []                       # 消し忘れ＝追加で消すべき孤児

while worklist が空でない:
    R = worklist.pop()               # DFS: 末尾から取る＝スタック
    for S in (R が参照していたシンボル):    # R の out-edge を辿る
        if S ∈ protected:  continue          # 枝刈り
        if S ∈ removed:    continue          # visited: 再訪しない → 必ず停止
        referrers = リポジトリ全体での S の参照箇所
        live = referrers − removed − staged  # 削除予定を除いた残存参照
        if live が空:                        # S は新規孤児
            staged.append(S)
            removed.add(S); worklist.push(S) # 深く潜る（連鎖）
report(staged)
```

`removed`（visited 集合）で各ノードを高々1回しか訪れないため**必ず停止**し、結果は起点集合に対する不動点になる。

## 手順

### 0. まずレンズを選ぶ
最低限の推奨シーケンス:

```
L4/L5（seedを網羅的に作る）→ L9（命名族で広く拾う）→ L1（前方DFSで孤児確定）
→ L2（後方で壊れた参照）→ L10（ツールで裏取り）→ L11/L12（実測で締め）
```

状況で追加: **置換・移行系なら L6** / **下流副作用が疑わしいなら L7** / **repo外に染み出すなら L8** / **seedの網羅性が不安なら L3**。

### footprint 軸表（L4 の実体）
削除対象を、以下の軸で漏れなく洗い出して seed 化する（UI だけ見ると衛星定義を取りこぼす）:

| 軸 | 目印 |
|---|---|
| A UI/表示 | component / style / story |
| B データ取得 | GraphQL query / fragment / operation |
| C ルーティング | routes 定義・payload 型 |
| D i18n | ロケール定義の葉キー、`t('a.b.c')` |
| E テストデータ | testdata 定数・import |
| F ログ/イベント計測 | `eventType: 'X'`、イベント型 export（**別ファイルに定義されがち＝漏れの本命**） |
| G 導線 | 他画面からのリンク・ボタン・遷移（→L2） |
| H リダイレクト | next.config / middleware の redirect・rewrite |
| I GraphQL スキーマ | BFF distributed schema / codegen 生成型 |
| J アセット | 画面固有の画像・イラスト・動画 |
| K feature flag | トグル・A/Bテスト・実験定義 |
| L E2E / VRT | playwright / スナップショット |
| M 型/定数/hook/util | コロケーション外の専用シンボル |
| N cookie/localStorage | 永続化キー |

### 1. 起点集合を差分から抽出（L4/L5/L9）
```bash
BASE=origin/main
git diff --diff-filter=D --name-only $BASE...HEAD     # 丸ごと削除されたファイル
git diff $BASE...HEAD -- <変更ファイル> | grep '^-'    # 変更ファイルから消えた行
git log --diff-filter=A -- <feature-path>             # L5: 導入コミット＝footprint台帳
```
消えた行/ファイルから、export・union メンバー・イベント型・i18n キー・ルート・GraphQL 名・testdata 定数・アセット import を洗い出す。命名族（L9）も併せて列挙。

### 2. 各シンボルの残存参照を数える（out-edge を辿る＝DFS 1 段）
削除対象ファイル群を `DEL` に列挙し、それを除外して全リポジトリ検索:
```bash
rg -n --fixed-strings "C" -- src \
  | rg -v -f <(printf '%s\n' "${DEL[@]}")
```
- ヒット 0 → **孤児候補**。定義が「変更ファイル（レジストリ）」側に残っていれば**それが削除漏れ**。
- ヒットが「定義行 1 箇所だけ」→ 実質孤児（自分自身しか参照していない）。**これが最頻の削除漏れパターン。**
- ヒットが実コードにある → **保護** or 対象外。残す。

### 3. 連鎖を辿る（DFS 本体）
孤児と判定したシンボルを `removed` に足し、**そのシンボルの定義が参照していた**別シンボルについて 2 を再帰。新規孤児が出なくなったら（不動点）終了。

### 4. 後方・ツール・実測で締める
- **L2**: 削除した機能を**指していた**参照（import/リンク/遷移）を grep し、壊れた参照が残らないか確認。
- **L10**: `npx knip` / `npx ts-prune` を通し、未使用 export・ファイルを裏取り（文字列キーは L1/L9 が担う）。
- **L11/L12**: route/bundle/codegen manifest に痕跡ゼロか、テレメトリ上まだ発火/被叩きがないかを照合。

## ガードレール（誤検知＝false negative を人に返す）

孤児「候補」は自動削除しない。以下は必ず人間に確認を促す:

- **動的参照**で grep が空振りしうるもの: 動的生成キー（`` t(`error.${code}`) ``、テンプレートで組むイベント名/ルート）、コード生成物（GraphQL codegen・reflection・DI）。
- **境界越え（L8）**: 設定ファイル・別リポジトリ（BFF resolver、分析ダッシュボード、ログ仕様）からの参照はリポジトリ内 grep では見えない。
- **意図的保留 vs 真の消し忘れ**: 「repo内では孤児だが、意図的に段階削除しているもの」を区別する。例: BFFフィールド廃止をバックエンドと別途調整予定にしている distributed schema（`*.graphqls`）。前者は「バグ」ではなく**要人間確認**として返す。

## 出力フォーマット

```
## 削除漏れ候補（新規孤児）
1. <シンボル> — 定義: <file:line> / 残存参照: 0（自定義のみ）/ 種別: <union|export|i18n|schema|...>
   → 消すべき: <file の該当箇所>
   経路(DFS/レンズ): <どのレンズ・どのseed削除から連鎖したか>

## 要人間確認（動的参照 / 境界越え / 意図的保留の疑い）
- <シンボル> — grep 0 だが <理由> のため保留

## 保護（枝刈り済み・残す）
- <共有資産シンボル> — <参照している現役箇所>
```

最後に「適用レンズ / 孤児確定数 / 保護数 / 要確認数」を1行サマリで出す。
