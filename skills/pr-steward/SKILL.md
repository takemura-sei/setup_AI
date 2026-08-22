---
name: pr-steward
description: PRを作成したら自動でアクティビティ監視（CI失敗・レビューコメント）に入り、「診断→修正→push」または「質問」のいずれかで必ず応答する運用をまとめたプロジェクト固有ドキュメント（`.claude/skills/steward/SKILL.md`）を生成する。「PR監視の仕組みを入れたい」「pr-stewardを使いたい」と言われたとき、または新規プロジェクトにPR自動追従の運用を導入したいときに使う。
---

# pr-steward

## 何をするものか

Claude Code は、PRを作成した後にCI失敗やレビューコメントへ追従する組み込みの運用ルール（PRアクティビティ監視・CI red対応・レビュー対応の判断フロー）を持っている。ただしそのルールは「マージ方針（merge/rebase）」「lockfileの再生成コマンド」「触ってはいけないファイル」「flakyテストの扱い」など、プロジェクトごとに異なる前提を必要とする。

本Skillは、対象プロジェクトの規約・CI構成・パッケージマネージャを検出し、それらの前提を埋めた `.claude/skills/steward/SKILL.md` を生成する。Claude Codeは組み込みの運用ルールを適用する際、リポジトリの `.claude/skills/steward/SKILL.md`（または `.claude/skills/babysit/SKILL.md`）を最優先の補足情報として読みに行くため、これを配置しておくだけで「issue駆動 + 1セッション1タスク」のフローに「PRを作ったら自動で監視状態に入り、CI失敗やレビューコメントに追従する」運用が組み込まれる。

## 呼び出しタイミング

- 「PR監視の仕組みを入れたい」「pr-stewardを使いたい」と言われたとき
- 新規プロジェクトのセットアップ時、または既存プロジェクトへ後からPR自動追従の運用を導入したいとき
- CI失敗対応やレビュー対応が都度手動になっていて、方針が明文化されていないことに気づいたとき

## 事前準備: プロジェクト前提の検出

生成前に、対象プロジェクトの以下を実際に確認する。要約や記憶に頼らず、都度ファイルを確認すること。

1. パッケージマネージャ（`package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` の有無）とlockfile再生成コマンド
2. `package.json` の `scripts`（lint / typecheck / test / build のうち存在するもの）や、それ以外の言語であればCI設定ファイル（`.github/workflows/*.yml` など）から、CI通過に必要な検証コマンドを洗い出す
3. デフォルトブランチ名とブランチ運用（自分が作ったブランチはrebase可か、merge固定か）。`CLAUDE.md` / `.claude/docs/development.md` があれば優先して確認する
4. 触ってはいけない・自動生成物として扱うファイル（生成済みlockfile、マイグレーション履歴など）
5. 上記が文書化されていない場合は、一般的なベストプラクティス（lockfileは手で編集せずコマンドで再生成する、flakyの断定は避ける、等）をデフォルト値として採用し、その旨を生成物内に明記する

## 生成対象ファイル

`.claude/skills/steward/SKILL.md`

## テンプレート構成

`templates/steward.md.template` … 生成するSKILL.md本体。プレースホルダーは以下の通り、単純文字列置換で埋める。

- `__PROJECT_NAME__` … 対象プロジェクト名
- `__VALIDATION_COMMANDS__` … CI通過前にローカルで走らせる検証コマンド一覧（lint / typecheck / test / build のうち存在するもの）
- `__LOCKFILE_REGEN_COMMAND__` … lockfileを再生成するコマンド（例: `npm install`, `pnpm install`）。存在しなければ「lockfileなし」と明記
- `__MERGE_POLICY__` … 自分が作成したブランチのbase取り込み方針（merge commit / rebase のどちらか。プロジェクトの規約が見つからなければ「merge commit（履歴を書き換えない）」をデフォルトにする）
- `__PROTECTED_PATHS__` … 自動修正の対象から外すパス（見つからなければ「特になし」）
- `__PROJECT_DOCS_PATHS__` … 規約が書かれているドキュメントのパス一覧（例: `CLAUDE.md`, `.claude/docs/development.md`）。存在しなければ「なし」

## 生成手順

1. 事前準備でプロジェクト前提を検出する
2. `templates/steward.md.template` を読み、プレースホルダーを置換する
3. `.claude/skills/steward/SKILL.md` として書き出す
4. 既に同名のファイルが存在する場合は上書きせずユーザーに確認する（破壊的操作のため）
5. 生成後、実際にPRを1つ作成し、CI失敗またはレビューコメントに対して「診断→修正→push」または「質問」で応答できることを試験運用で確認する

## ユーザー確認が必要な操作

- 既存の `.claude/skills/steward/SKILL.md` への上書き
- プロジェクトのマージ方針・保護対象パスが検出できず、デフォルト値を採用するかどうかの判断

## 制約・注意点

- 本Skill自身はPRを監視しない。あくまで「Claude Codeの組み込みPR監視ルールが参照する、プロジェクト固有の補足ドキュメント」を生成するだけで、実際の監視・応答はPR作成後にClaude Codeが自動的に行う
- PRアクティビティの購読（CI・レビューコメントのイベント通知）自体はホスト環境（Claude Code on the web／CLI）の機能に依存する。購読の仕組みを提供しない環境では、`loop` Skillなどで定期的にPR状態を確認する運用に読み替える
- 生成物はあくまで「前提条件の申し送り」であり、CI red・レビュー対応の判断ロジック自体（診断→修正→pushか質問か、flakyの扱い等）はClaude Code側の組み込みルールに従う。生成物でその判断ロジックを上書きしようとしない

## 使用例

```
pr-stewardでPR監視の運用を入れて
```

→ 対象プロジェクトのCI構成・パッケージマネージャ・マージ方針を検出し、`.claude/skills/steward/SKILL.md` が生成される。以降このプロジェクトでPRを作成すると、Claude Codeが自動でアクティビティ監視に入り、CI失敗やレビューコメントに追従するようになる。
