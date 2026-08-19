---
name: feature-scaffold
description: Nuxtレイヤードアーキテクチャ（型 → server/api → service → store → composable → component → page → test）に沿って、新機能の雛形ファイル一式をTODOコメント付きで生成する。「〇〇機能を追加したい」「新機能の雛形を作って」と言われたとき、ゼロから新しいドメイン機能を実装し始めるときに使う。
---

# feature-scaffold

## 何をするものか

機能名（例: `game`）を受け取り、Nuxtのレイヤードアーキテクチャの各層に沿った雛形ファイルを一括生成するSkill。`pokemon_apply_v2` の `.claude/docs/adding-a-feature.md` にある9段階の定型手順（型 → server/api → service → store → composable → component → page → test）を毎回手作業でなぞる代わりに、抜け漏れなく生成する。

## 呼び出しタイミング

- 新しいドメイン機能をゼロから追加し始めるとき
- 「〇〇機能のひな形を作って」「feature-scaffoldして」と言われたとき

## 事前準備: プロジェクト構成の検出

生成前に必ず以下を確認し、実ディレクトリ名（`<root>`）を決定する。生成先を誤ると大量の誤配置ファイルができてしまうため、自動検出できない場合は必ずユーザーに確認する。

1. `nuxt.config.ts`（または `.js`）の `srcDir` 設定を読む。指定があればそれを `<root>` とする。
2. 指定がなければ、リポジトリ直下に `app/` と `src/` のどちらが存在するか確認する。
   - Nuxt 3のデフォルトは `srcDir: '.'`（リポジトリ直下に `server/`, `pages/` 等）だが、プロジェクトによっては `app/` 配下にまとめている場合がある。
3. どちらとも判断できない場合は、ユーザーに確認する。

## 生成対象ファイル

機能名 `<feature>`（例: `game`）に対して、以下を生成する。

| 層 | パス | テンプレート |
|---|---|---|
| 型 | `<root>/shared/types/<feature>.ts` | `templates/type.ts.template` |
| server/api | `<root>/server/api/<feature>/index.get.ts` | `templates/server-api.get.ts.template` |
| service | `<root>/services/<feature>Service.ts` | `templates/service.ts.template` |
| store | `<root>/stores/<feature>.ts` | `templates/store.ts.template` |
| composable | `<root>/composables/use<Feature>.ts` | `templates/composable.ts.template` |
| component | `<root>/components/<Feature>/<Feature>.vue` | `templates/component.vue.template` |
| page | `<root>/pages/<feature>/index.vue` | `templates/page.vue.template` |
| test | `<root>/tests/<層に応じたミラーパス>/<feature>.spec.ts` | `templates/test.spec.ts.template` |

テスト配置は `src`（または `app`）の構造をミラーする方針とし、対象層ごとに置き場所を変える（例: service のテストは `tests/services/<feature>Service.spec.ts`）。

## プレースホルダー

テンプレート内のプレースホルダーは以下の2種類のみ。置換時はこのトークンを機能名から生成した文字列で単純置換する。

- `__FEATURE_PASCAL__` … パスカルケース（例: `game` → `Game`）
- `__FEATURE_CAMEL__` … キャメルケース（例: `game` → `game`、`game-log` → `gameLog`）

`{{ }}` 形式ではなく `__X__` 形式を採用しているのは、Vueのテンプレート補間構文 `{{ }}` と衝突しないようにするため（`component.vue.template` / `page.vue.template` は実際に `{{ }}` を使用する）。

## 生成手順

1. 事前準備でプロジェクト構成を検出する
2. 各層のファイルが既に存在するか確認する。**既存ファイルがあれば上書きせず、ユーザーに確認する**（破壊的操作のため）
3. `templates/` 配下の各テンプレートを読み、プレースホルダーを機能名で置換して該当パスに生成する
4. 生成後、`npm run lint` / `npm run typecheck` があれば実行し、雛形自体がエラーなく通ることを確認する
5. 生成したファイル一覧をユーザーに提示し、次のアクション（型定義の中身を埋める、API実装を進める等）を明示する

## ユーザー確認が必要な操作

- 既存ファイルへの上書き
- プロジェクト構成（`src/` か `app/` か等）が自動検出できず、判断が必要なとき

## 制約・注意点

- このSkillは「空の雛形＋TODOコメント」を作るだけで、ビジネスロジックの実装は行わない
- プロジェクトごとにディレクトリ命名やレイヤー構成、状態管理ライブラリ（Pinia以外の場合もある）が異なるため、`templates/` の内容はプロジェクトの実態に合わせて調整して使うことを想定する（万能テンプレートではない）
- 本Skillのテンプレートはこのリポジトリ内で作成したものであり、実プロジェクト（`pokemon_apply_v2` 等）での動作確認・生成物のパターン一致確認は未実施。実際に使う際は生成物を実プロジェクトの規約と突き合わせて調整すること

## 使用例

```
feature-scaffoldで game 機能の雛形を作って
```

→ `game` 機能の型・API・service・store・composable・component・page・testの雛形が生成される。
