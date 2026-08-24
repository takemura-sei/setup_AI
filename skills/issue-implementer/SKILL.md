---
name: issue-implementer
description: GitHub issueを渡すとテスト先行実装（`tests/` 配下、`src/` の構造をミラー）→実装→lint/typecheck/testの実行→コミットまでを担当するサブエージェント定義（`.claude/agents/issue-implementer.md`）を生成する。「issue実装用のサブエージェントを作って」「issue-implementerを使いたい」と言われたとき、または既存プロジェクトにissue駆動+TDDの実装フローを後付けしたいときに使う。
---

# issue-implementer

## 何をするものか

「issue駆動 + 1セッション1タスク + TDD」のワークフローを、issue番号を渡すだけで一貫して担当するサブエージェント定義を生成するSkill。生成されるサブエージェントは、issueの内容を読み取り、対応するテストファイルの作成（`tests/` 配下、`src/` の構造をミラー）→実装→検証コマンド（lint / typecheck / test）の実行→コミットまでを行う。「1 issue = 2〜3時間」程度のスコープの粒度を守り、issueに書かれていない変更には手を出さない。

汎用的にバグを探す `code-review` スキルや規約準拠をチェックする `code-reviewer` スキルとは異なり、本Skillが生成するサブエージェントは「実装そのもの」を担当する。レビューは別途 `code-review` / `code-reviewer` に委ねる想定。

## 呼び出しタイミング

- 「issue実装用のサブエージェントを作って」「issue-implementerを使いたい」と言われたとき
- 新規プロジェクトのセットアップ時、または既存プロジェクトへ後からissue駆動+TDDの実装フローを導入したいとき
- issue対応のたびに「テスト作成→実装→検証コマンド実行」の手順を毎回自力でなぞっていることに気づいたとき

## 事前準備: プロジェクト規約の検出

生成前に、対象プロジェクトの以下を実際に確認する。要約や記憶に頼らず、都度ファイルを確認すること。

1. `CLAUDE.md` や `.claude/docs/development.md` 等を読み、以下が明文化されていないか確認する
   - テストファイルの配置規則（`tests/` が `src/` の構造をミラーする、命名規則など）
   - TDDの運用（テスト先行かどうか、テストの粒度）
   - 「1 issue」のスコープの目安（時間・変更範囲）
   - コミットメッセージの規約
2. `package.json` の `scripts` を確認し、`lint` / `typecheck` / `test` のうち存在するコマンドを洗い出す（`ci-setup` / `pr-steward` Skillと同じ検出方法。名前が `type-check` 等の別名でも実質typecheckなら拾う）
3. 上記が文書化されていない場合は、一般的なベストプラクティス（`src/foo/bar.ts` に対して `tests/foo/bar.test.ts` を作る、1 issueは焦点を絞った差分に留める、等）をデフォルト値として採用し、その旨を生成物内に明記する

## 生成対象ファイル

`.claude/agents/issue-implementer.md`

## テンプレート構成

`templates/issue-implementer.md.template` … サブエージェント定義本体（Frontmatter + システムプロンプト）。プレースホルダーは以下の通り、単純文字列置換で埋める。

- `__PROJECT_DOCS_PATHS__` … 規約が書かれているドキュメントのパス一覧（例: `CLAUDE.md`, `.claude/docs/development.md`）。サブエージェントが実行時にも一次情報として参照できるよう、パスをそのまま埋め込む
- `__TEST_STRUCTURE__` … テストファイルの配置規則の要約（検出できなければ「`src/` の構造を `tests/` にミラーする」をデフォルトとして明記）
- `__VALIDATION_COMMANDS__` … 実装後にローカルで実行する検証コマンド一覧（lint / typecheck / test のうち存在するもの）
- `__SCOPE_GUIDELINE__` … 「1 issue」のスコープの目安（検出できなければ「issueに書かれていない変更には手を出さず、スコープが大きいと判断した場合は実装を進めずユーザーに分割を提案する」をデフォルトとする）

## 生成手順

1. 事前準備でプロジェクト規約を検出する
2. `templates/issue-implementer.md.template` を読み、`__PROJECT_DOCS_PATHS__` / `__TEST_STRUCTURE__` / `__VALIDATION_COMMANDS__` / `__SCOPE_GUIDELINE__` を置換する
3. `.claude/agents/issue-implementer.md` として書き出す
4. 既に同名のファイルが存在する場合は上書きせずユーザーに確認する（破壊的操作のため）
5. 生成後、実際の小さいissueに対してサブエージェントを呼び出し、テスト作成→実装→検証コマンド実行→コミットの流れが回ることを確認する

## ユーザー確認が必要な操作

- 既に存在する `.claude/agents/issue-implementer.md` への上書き
- テスト配置規則・スコープの目安が検出できず、デフォルト値を採用するかどうかの判断
- 生成されたサブエージェントによるコミットの実行（サブエージェント自身がpushやPR作成まで行うかどうかは、呼び出し側の運用に委ねる）

## 制約・注意点

- 本Skill自身はissueを実装しない。あくまで「issueを実装するサブエージェント定義」を生成するだけで、実際の実装は生成された `issue-implementer` サブエージェントを呼び出して行う
- 検出できるのはドキュメント化された規約と `package.json` の `scripts` のみ。暗黙のテスト配置慣習までは拾えない
- 生成されるサブエージェントは「issueに書かれた範囲」を実装対象とする。issueの記述が曖昧、または対応中にスコープ外の変更が必要だと判明した場合は、実装を進めずユーザーに確認する設計にしている
- push・PR作成・マージ判断は本Skillの生成物の対象外。それらの運用は `pr-steward` Skillなど別の仕組みに委ねる

## 使用例

```
issue-implementerで実装用サブエージェントを作って
```

→ 対象プロジェクトのテスト配置規則・検証コマンド・スコープの目安を検出し、`.claude/agents/issue-implementer.md` が生成される。以降 `issue #42 を issue-implementer で実装して` のように呼び出すと、テスト作成→実装→検証コマンド実行→コミットまでを担当する。
