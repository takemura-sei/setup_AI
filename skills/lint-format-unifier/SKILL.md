---
name: lint-format-unifier
description: プロジェクトの用途・規模を簡単にヒアリングしESLint / Biomeいずれかを選定した上で、設定ファイル（config）・`package.json` のスクリプト・`.claude/settings.json` のallow項目を一括で配置する。「lintを整備して」「ESLintかBiomeか決めて設定して」「新規プロジェクトのlint/format周りをセットアップして」と言われたとき、または `claude-setup` / `project-bootstrap` の一部として使う。
---

# lint-format-unifier

## 何をするものか

プロジェクトごとにESLint（`pokemon_apply_v2`, `typing-game` 等）とBiome（`front-test` 等）が混在し、新規プロジェクト作成のたびにどちらを使うか都度判断している状況に対応するSkill。ユーザーへの簡単なヒアリングでESLint / Biomeいずれかを選定し、以下3点をまとめて配置する。

1. linter/formatterの設定ファイル（`eslint.config.mjs` または `biome.json`）
2. `package.json` の `scripts`（`lint` / `lint:fix` / `format` 等）
3. `.claude/settings.json` の `permissions.allow` 項目（生成したスクリプトをClaude Codeが確認なしで実行できるようにする許可リスト）

`claude-setup` / `project-bootstrap` から部品として呼び出されることを想定しているが、既存プロジェクトへの後付け導入にも単体で使える。

## 呼び出しタイミング

- 「lintを整備して」「ESLintかBiomeで設定して」と言われたとき
- 新規プロジェクトのセットアップ（`project-bootstrap` 等）の一環として
- 既存プロジェクトでESLint/Biomeの設定が無い、またはlint/format方針が定まっていないことに気づいたとき

## 事前準備: ヒアリングとプロジェクト構成の検出

### 1. ESLint / Biome の選定ヒアリング

会話やリポジトリから既に判断材料がある場合は聞き直さず、無ければ以下を簡潔に確認する。

- Vue/Nuxtプロジェクトか（`eslint-plugin-vue` によるVueテンプレート内静的解析など、Vue系エコシステムの充実度が要るか）
- 既存のフォーマッタ（Prettier等）を使い続けたいか、lint/formatを1ツールに統合したいか
- チームの慣れているツールに強い指定があるか

判断に迷う場合の目安（ユーザーに提示してよい）:

| 観点 | ESLint | Biome |
|---|---|---|
| Vue/Nuxtのテンプレート解析 | `eslint-plugin-vue` で対応、実績が多い | 対応は発展途上 |
| セットアップの速さ・依存の少なさ | プラグイン構成で依存が増えがち | 単一バイナリで完結、高速 |
| 既存プロジェクトとの一貫性 | `pokemon_apply_v2` / `typing-game` と揃う | `front-test` と揃う |

最終的な選定はユーザーの回答を優先する。

### 2. プロジェクト構成の検出

1. `package.json` を読み、`dependencies` / `devDependencies` に `vue` または `nuxt` があるか確認する（ESLint選定時に `eslint-plugin-vue` を組み込むかどうかの判断に使う）
2. `package.json` に既に `lint` / `format` 系のスクリプトが定義されていないか確認する
3. 既に `eslint.config.mjs`（または `.eslintrc*`）や `biome.json` が存在する場合は、上書きせずユーザーに確認する（破壊的操作のため）
4. `.claude/settings.json` が既に存在する場合は、`permissions.allow` に追記する形にする（ファイル全体を上書きしない）

## 生成対象ファイル

- `eslint.config.mjs`（ESLint選定時）または `biome.json`（Biome選定時）
- `package.json` の `scripts` への追記
- `.claude/settings.json` の `permissions.allow` への追記

## テンプレート構成

`templates/eslint/` と `templates/biome/` に、選定したツールごとの断片を用意している。プレースホルダーは単純文字列置換で埋める。

| ファイル | 役割 |
|---|---|
| `templates/eslint/eslint.config.mjs.template` | ESLint flat config本体。`// __VUE_IMPORT__` / `// __VUE_CONFIG__` の行がVue対応の差し込みプレースホルダー |
| `templates/eslint/package-scripts.json.template` | `package.json` に追記する `scripts` の断片（lint系） |
| `templates/eslint/settings-allow.json.template` | `.claude/settings.json` の `permissions.allow` に追記する断片 |
| `templates/biome/biome.json.template` | Biome設定本体。`__BIOME_VERSION__` が `$schema` のバージョンプレースホルダー |
| `templates/biome/package-scripts.json.template` | `package.json` に追記する `scripts` の断片（lint/format系） |
| `templates/biome/settings-allow.json.template` | `.claude/settings.json` の `permissions.allow` に追記する断片 |

## 生成手順

### ESLintを選定した場合

1. `templates/eslint/eslint.config.mjs.template` を読む
2. Vue/Nuxtプロジェクトの場合、`// __VUE_IMPORT__` を `import vue from 'eslint-plugin-vue'` に、`// __VUE_CONFIG__` を `...vue.configs['flat/recommended'],` に置換する。Vue/Nuxtでない場合は両プレースホルダー行を削除する
3. `eslint.config.mjs` として書き出す
4. `templates/eslint/package-scripts.json.template` の内容を、既存の `package.json` の `scripts` にキーの重複が無いか確認しながらマージする（既存の同名スクリプトは上書きせずユーザーに確認する）
5. `templates/eslint/settings-allow.json.template` の `permissions.allow` の項目を、既存の `.claude/settings.json`（無ければ新規作成）にマージする
6. 必要な依存関係（`eslint`, `typescript-eslint`, `@eslint/js`, Vue/Nuxtの場合は `eslint-plugin-vue` も）をdevDependenciesとして追加する必要があることをユーザーに伝える（インストールコマンドの実行そのものは、パッケージマネージャの実行を伴う操作のためユーザー確認を挟む）

### Biomeを選定した場合

1. `templates/biome/biome.json.template` を読み、`__BIOME_VERSION__` を利用するBiomeのバージョン（不明な場合は最新の安定版）に置換する
2. `biome.json` として書き出す
3. `templates/biome/package-scripts.json.template` の内容を、既存の `package.json` の `scripts` にマージする（手順はESLintと同様）
4. `templates/biome/settings-allow.json.template` の `permissions.allow` の項目を、既存の `.claude/settings.json` にマージする
5. 依存関係 `@biomejs/biome` をdevDependenciesとして追加する必要があることをユーザーに伝える

### 共通

6. 生成・マージした内容をユーザーに提示し、`npm run lint`（または選定したツールのコマンド）が実際に通るか確認するよう促す

## ユーザー確認が必要な操作

- 既存の `eslint.config.mjs` / `.eslintrc*` / `biome.json` への上書き
- `package.json` の既存スクリプトとキーが重複する場合の上書き
- devDependenciesの追加インストール（パッケージマネージャコマンドの実行）
- ESLint / Biomeの最終選定がヒアリングだけで決めきれない場合の判断

## 制約・注意点

- 本Skillは設定ファイル一式の配置までを行う。実際に `npm install` を実行して依存関係を解決するかどうかはユーザー判断に委ねる
- モノレポ構成（複数の `package.json` が存在する等）の場合は、対象パッケージをユーザーに確認したうえで配置先を調整する必要がある。本Skillは単一パッケージのリポジトリを主対象としている
- 生成するESLint設定はflat config形式（ESLint v9+）を前提とする。レガシーな `.eslintrc*` 形式のプロジェクトに適用する場合は移行が必要になる旨をユーザーに伝える
- Prettier等、既存の別フォーマッタと併用したい場合の共存設定（`eslint-config-prettier` の要否など）は対象外。ヒアリング時に既存フォーマッタの有無を確認し、必要であれば別途調整が要ることを申し送りする

## 使用例

```
lint-format-unifierでlintを整備して
```

→ Vue/Nuxtプロジェクトかどうかのヒアリング後、ESLintかBiomeいずれかが選定され、`eslint.config.mjs`（または `biome.json`）・`package.json` の `scripts`・`.claude/settings.json` の `permissions.allow` がまとめて配置される。
