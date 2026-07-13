# template

[![prek](https://github.com/dope-corp/template/actions/workflows/prek.yaml/badge.svg)](https://github.com/dope-corp/template/actions/workflows/prek.yaml)
[![gitleaks](https://github.com/dope-corp/template/actions/workflows/gitleaks.yaml/badge.svg)](https://github.com/dope-corp/template/actions/workflows/gitleaks.yaml)
[![mise-lock](https://github.com/dope-corp/template/actions/workflows/mise-lock.yaml/badge.svg)](https://github.com/dope-corp/template/actions/workflows/mise-lock.yaml)

dope-corp organization の新規リポジトリ作成時に使用する共通テンプレートです。

新規リポジトリを作成する際、「Repository template」としてこのリポジトリを選択すると、以下のファイルが引き継がれます。

- `.gitignore`
- `.github/ISSUE_TEMPLATE/bug_report.md`
- `.github/ISSUE_TEMPLATE/feature_request.md`
- `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/CODEOWNERS`
- `.github/workflows/prek.yaml`
- `.github/workflows/gitleaks.yaml`
- `.github/workflows/mise-lock.yaml`
- `.github/workflows/claude.yaml`
- `mise.toml` / `mise.lock`
- `.pre-commit-config.yaml`
- `dprint.json`
- `renovate.json`

各リポジトリの事情に合わせて、生成後に内容を編集してください。
引き継がれるのは**ファイルのみ**で、リポジトリ設定・ruleset は引き継がれないため、
生成後に「[GitHub リポジトリ設定](#github-リポジトリ設定)」の初期設定を行ってください。

`.gitignore` はホワイトリスト方式 (`*` で全て無視し、`!` で許可したものだけを追跡する) になっている。
新しく追跡したいファイルを追加する場合は、対応する `!` の行を追記する必要がある。

## セットアップ

このリポジトリは [mise](https://mise.jdx.dev/) の利用を前提としています。

```sh
mise trust    # 初回のみ: このディレクトリの mise.toml を信頼する
mise install  # tools (prek 等) をインストールし、pre-commit hook をセットアップする
```

`mise install` を実行すると `[hooks] postinstall` により `prek install` が自動実行され、
`.git/hooks/pre-commit` がセットアップされる。

pre-commit hook には `dprint-fmt` (`.pre-commit-config.yaml` 内) が含まれており、
json/markdown/toml/yaml ファイルは [dprint](https://dprint.dev/) (`dprint.json` で設定) により
自動フォーマットされる。dprint plugin の WASM URL は `url@sha256` の checksum 付きで pin されており、
バージョンと checksum は Renovate (`renovate.json` の customManagers) が追従させる。

## CI

- `.github/workflows/prek.yaml`: push / pull request 時に、`mise.lock` 通りのツールで
  `.pre-commit-config.yaml` の全フックを `prek run --all-files` で実行する。
- `.github/workflows/gitleaks.yaml`: push / pull request 時に、コミット履歴全体を対象に
  [gitleaks](https://github.com/gitleaks/gitleaks) でシークレットスキャンを行う。
  検知ルールの更新を取り込むため、毎週月曜に main の履歴全体を再スキャンする。
- `.github/workflows/mise-lock.yaml`: `mise.toml` に pin されたバージョンのまま `mise lock` を実行し、
  `mise.lock` の checksum/URL を最新化する。`mise.toml` のバージョン自体の更新は Renovate に任せる。
  - `pull_request` (`mise.toml` を変更する PR、主に Renovate が対象):
    同じ PR のブランチに直接 commit して追従させる。
  - 毎週月曜 (`schedule`) / `workflow_dispatch`: 差分があれば `chore/mise-lock` ブランチで PR を作成する。
  - `GITHUB_TOKEN` による push / PR 作成からは workflow run が発火しない (再帰実行防止の GitHub 仕様) ため、
    更新したブランチへ prek / gitleaks を `workflow_dispatch` で明示的に起動して required check を付ける。
    ci.yaml を追加したリポジトリでは dispatch 対象に ci.yaml も加えること。

すべての workflow に `concurrency` を設定しており、同一 ref で新しい run が始まると
実行中の古い run はキャンセルされる (変更を push しうる claude / mise-lock を除く)。
ワークフローの外部依存 (`uses:` / `container:`) は commit SHA またはイメージ digest で固定し、
バージョンをコメントで併記する (digest の更新も Renovate が行う)。

build / test を持つリポジトリでは、prek が扱わない `npm audit` / test / build のみを行う
ci.yaml (job 名は required check に合わせて `verify`) を追加する。参照実装:
[dope-corp/web-app の ci.yaml](https://github.com/dope-corp/web-app/blob/main/.github/workflows/ci.yaml)。

## Claude Code (claude.yaml)

`.github/workflows/claude.yaml` は、issue / PR コメント等の `@claude` メンションで
[claude-code-action](https://github.com/anthropics/claude-code-action) を起動する
(author_association による一次フィルタ付き。書き込み権限の厳密な検証は action 内部で行われる)。

実行には secret `CLAUDE_CODE_OAUTH_TOKEN` が必要だが、**dope-corp では organization レベルの
secret (visibility: all) として設定済みのため、organization 内のリポジトリでは追加設定は不要**。
organization 外へファイルを流用する場合や、リポジトリ単位でトークンを分けたい場合のみ以下を実行する
(リポジトリ secret は organization secret より優先される)。

```sh
claude setup-token   # Claude Code の OAuth トークンを発行
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <owner>/<repo>
```

## node を利用する場合

Node.js を使うリポジトリでは、既存の統一済みリポジトリ (dope-homepage / web-app 等) と同じ規約に揃える。

- バージョンの単一ソースとして `.node-version` を作成し、**メジャーのみ** (例: `26`) を書く。
  `.nvmrc` は作らない。Cloudflare のビルドや各種ツールもこのファイルを参照する。
- `mise.toml` の `[settings]` に `idiomatic_version_file_enable_tools = ["node"]` を追加し、
  mise にも `.node-version` を読ませる。
- `.github/workflows/mise-lock.yaml` の `pull_request` トリガーの `paths` に `.node-version` を
  追加する (バージョン変更 PR に `mise.lock` を追従させるため)。
- パッチバージョンの追従は mise-lock.yaml の週次実行が `mise.lock` の再解決で行い、
  メジャー更新の提案は Renovate (nodenv manager、デフォルトで有効) が行う。
- `.gitignore` (ホワイトリスト方式) に `!/.node-version` を追記する。
- ci.yaml / eslint / typecheck / commitlint / dprint (language: node +
  additional_dependencies) の pre-commit hook 構成は
  [dope-corp/web-app](https://github.com/dope-corp/web-app) を参照実装とする。

## GitHub リポジトリ設定

### organization レベルで設定済みのもの (リポジトリ側の作業不要)

- Actions の許可ポリシー: `allowed_actions: selected`。GitHub 公式 / verified creator に加え、
  `patterns_allowed` で `jdx/mise-action@*` と `anthropics/claude-code-action@*` を許可。
  外部 action の SHA pin は必須 (`sha_pinning_required: true`)。
- org ruleset「Protect main branch」: 全リポジトリの main が対象。PR 必須・squash merge 限定・
  ブランチ削除禁止・linear history。org レベルの ruleset なので**個別リポジトリから変更しないこと**。
- org secret `CLAUDE_CODE_OAUTH_TOKEN` (visibility: all)。

### リポジトリ生成後に初回に行う設定

2026-07 に全リポジトリへ統一適用した設定。テンプレートからは引き継がれないため、生成のたびに実行する。

```sh
REPO=dope-corp/<repo>

# merge 方式: squash のみ / squash タイトルは COMMIT_OR_PR_TITLE /
# merge 後にブランチ自動削除 / wiki off
gh api -X PATCH "repos/$REPO" --input - <<'JSON'
{
  "allow_merge_commit": false,
  "allow_rebase_merge": false,
  "allow_squash_merge": true,
  "squash_merge_commit_title": "COMMIT_OR_PR_TITLE",
  "squash_merge_commit_message": "COMMIT_MESSAGES",
  "delete_branch_on_merge": true,
  "has_wiki": false
}
JSON

# main への merge に CI green を必須化する repo ruleset。
# required checks は workflow 構成に合わせる: ci.yaml (job: verify) を追加したら
# {"context": "verify", "integration_id": 15368} も加える。
gh api -X POST "repos/$REPO/rulesets" --input - <<'JSON'
{
  "name": "Require CI green on main",
  "target": "branch",
  "enforcement": "active",
  "bypass_actors": [],
  "conditions": {"ref_name": {"include": ["refs/heads/main"], "exclude": []}},
  "rules": [
    {"type": "required_status_checks", "parameters": {
      "do_not_enforce_on_create": false,
      "strict_required_status_checks_policy": false,
      "required_status_checks": [
        {"context": "prek", "integration_id": 15368},
        {"context": "gitleaks", "integration_id": 15368}
      ]
    }}
  ]
}
JSON
```

secret scanning (push protection / non-provider patterns / validity checks) が有効になっているかを
Settings → Advanced Security で確認する (org の新規リポジトリ既定で有効化されていない場合は個別に有効化する)。

## 環境変数

- ディレクトリ単位の環境変数は `mise.toml` の `[env]` に定義するか、`.env` ファイルを `_.file` で読み込む。
- シェル起動時に環境変数・PATH を自動反映させるため、シェルの rc ファイルに以下を追加する。

  ```sh
  # ~/.zshrc (bash の場合は `mise activate bash`)
  eval "$(mise activate zsh)"
  ```

## タスク

タスクは `mise run <task>` で実行する。`mise.toml` の `[tasks]` を変更した場合は
`mise run docs` を実行し、以下の一覧を更新する (pre-commit hook からも自動実行される)。

<!-- dprint-ignore-start -->
<!-- mise-tasks -->
## `docs`

- **Usage**: `docs`

Sync the task list embedded in README.md with mise.toml
<!-- /mise-tasks -->
<!-- dprint-ignore-end -->
