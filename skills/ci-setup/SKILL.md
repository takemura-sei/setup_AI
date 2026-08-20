---
name: ci-setup
description: package.json の scripts（lint / typecheck / test / build）を検出し、PR時に自動実行するGitHub Actionsワークフロー（`.github/workflows/ci.yml`）を生成する。「CIを整備して」「GitHub Actionsを設定して」「既存プロジェクトにCIを追加して」と言われたとき、または新規プロジェクトのセットアップ時に使う。
---

# ci-setup

## 何をするものか

`npm run lint` / `npm run typecheck` / `npm run test` / `npm run build` をPR作成・更新時に自動実行するGitHub Actionsワークフローを生成するSkill。`package.json` の `scripts` を検出し、**存在するコマンドだけ**組み込む（例: `typecheck` スクリプトが無ければそのステップ自体を省略する）。

`claude-setup` / `project-bootstrap` スキルの延長として、新規プロジェクトだけでなく既存プロジェクトへも後から追加できる形にしている。

## 呼び出しタイミング

- 「CIを整備して」「GitHub Actionsでlint/testを回して」と言われたとき
- 新規プロジェクトのセットアップ（`project-bootstrap` 等）の一環として
- `.github/workflows` が存在しない、またはlint/typecheck/testがローカル実行のみのプロジェクトに気づいたとき

## 事前準備: プロジェクト構成の検出

生成前に必ず以下を確認する。

1. `package.json` を読み、`scripts` に `lint` / `typecheck` / `test` / `build` のいずれが定義されているか確認する。定義されているものだけをワークフローに組み込む対象とする。
   - `typecheck` という名前ではなく `type-check` / `tsc` 等の別名で定義されているプロジェクトもあるため、`scripts` の値（実行内容）も確認し、実質的にtypecheckを行っているスクリプトがあれば拾う。
2. パッケージマネージャを検出する。ロックファイルの有無で判定する。
   - `package-lock.json` → npm（`npm ci` / `npm run <script>`）
   - `yarn.lock` → yarn（`yarn install --frozen-lockfile` / `yarn <script>`）
   - `pnpm-lock.yaml` → pnpm（`pnpm install --frozen-lockfile` / `pnpm run <script>`、setup-nodeの前に `pnpm/action-setup` が必要）
   - 判定できない場合はnpmをデフォルトとし、ユーザーに確認する。
3. Node.jsのバージョンを検出する。`.nvmrc` / `package.json` の `engines.node` があればそれを使う。無ければ `20` をデフォルトとする。
4. 既に `.github/workflows/ci.yml`（または類似のワークフロー）が存在する場合は、上書きせずユーザーに確認する（破壊的操作のため）。

## 生成対象ファイル

`.github/workflows/ci.yml`

## テンプレート構成

`templates/` 配下に、ワークフローの骨組みと各チェック項目のステップ断片を分けて用意している。

| ファイル | 役割 |
|---|---|
| `templates/base.yml.template` | ワークフロー全体の骨組み（トリガー、checkout、Node.jsセットアップ、依存インストール）。`# __STEPS__` の行に各ステップ断片を差し込む |
| `templates/step-lint.yml.template` | `lint` スクリプト用ステップ |
| `templates/step-typecheck.yml.template` | `typecheck` スクリプト用ステップ |
| `templates/step-test.yml.template` | `test` スクリプト用ステップ |
| `templates/step-build.yml.template` | `build` スクリプト用ステップ |

各テンプレートのプレースホルダーは以下の通り。単純文字列置換で埋める。

- `__NODE_VERSION__` … 検出したNode.jsバージョン（例: `20`）
- `__INSTALL_COMMAND__` … 検出したパッケージマネージャのインストールコマンド（例: `npm ci`）
- `__RUN_PREFIX__` … スクリプト実行コマンドの接頭辞（例: npmなら `npm run`、yarnなら `yarn`、pnpmなら `pnpm run`）

## 生成手順

1. 事前準備でプロジェクト構成（scripts・パッケージマネージャ・Node.jsバージョン）を検出する
2. `templates/base.yml.template` を読み、`__NODE_VERSION__` / `__INSTALL_COMMAND__` を置換する
   - pnpmの場合は `pnpm/action-setup` のステップも追加する（base側にコメントで挿入位置を明記している）
3. 検出した `scripts`（lint / typecheck / test / build）に対応する `templates/step-*.yml.template` を、この順序で読み、`__RUN_PREFIX__` を置換したうえで `base.yml.template` の `# __STEPS__` 行に差し込む。存在しないスクリプトの断片は挿入しない
4. 組み立てた内容を `.github/workflows/ci.yml` として書き出す
5. YAMLとして構文が壊れていないか目視確認する（インデント崩れに注意。ステップ断片は2スペース単位でインデントを揃えて連結する）
6. 生成後、ユーザーにワークフローの内容とトリガー条件（デフォルトは `pull_request` + `push` to デフォルトブランチ）を提示し、リポジトリの実際のデフォルトブランチ名（`main` / `master` 等）に合わせて調整が必要か確認する

## ユーザー確認が必要な操作

- 既存の `.github/workflows/ci.yml`（または類似ファイル）への上書き
- パッケージマネージャ・Node.jsバージョンが自動検出できず判断が必要なとき

## 制約・注意点

- 検出できるのは `scripts` に定義されたコマンドの有無まで。スクリプトの中身（例: `lint` が本当にlintを実行しているか）までは検証しない
- モノレポ構成（複数の `package.json` が存在する等）の場合は、対象パッケージをユーザーに確認したうえでワークフローのワーキングディレクトリを調整する必要がある。本Skillは単一パッケージのリポジトリを主対象としている
- デプロイやリリース関連のジョブは対象外。あくまで lint / typecheck / test / build の検証のみを行う

## 使用例

```
ci-setupでCIワークフローを作って
```

→ `package.json` の `scripts` を検出し、存在するチェック項目（例: lint と test のみ、typecheckは無し）だけを含んだ `.github/workflows/ci.yml` が生成される。
