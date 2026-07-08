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
- `mise.toml` / `mise.lock`
- `.pre-commit-config.yaml`
- `renovate.json`

各リポジトリの事情に合わせて、生成後に内容を編集してください。

## セットアップ

このリポジトリは [mise](https://mise.jdx.dev/) の利用を前提としています。

```sh
mise trust    # 初回のみ: このディレクトリの mise.toml を信頼する
mise install  # tools (prek 等) をインストールし、pre-commit hook をセットアップする
```

`mise install` を実行すると `[hooks] postinstall` により `prek install` が自動実行され、
`.git/hooks/pre-commit` がセットアップされる。

## CI

- `.github/workflows/prek.yaml`: push / pull request 時に、`mise.lock` 通りのツールで
  `.pre-commit-config.yaml` の全フックを `prek run --all-files` で実行する。
- `.github/workflows/gitleaks.yaml`: push / pull request 時に、コミット履歴全体を対象に
  [gitleaks](https://github.com/gitleaks/gitleaks) でシークレットスキャンを行う。
- `.github/workflows/mise-lock.yaml`: `mise.toml` に pin されたバージョンのまま `mise lock` を実行し、
  `mise.lock` の checksum/URL を最新化する。`mise.toml` のバージョン自体の更新は Renovate に任せる。
  - `pull_request` (`mise.toml` を変更する PR、主に Renovate が対象): 同じ PR のブランチに直接 commit して追従させる。
  - 毎週月曜 (`schedule`) / `workflow_dispatch`: 差分があれば `chore/mise-lock` ブランチで PR を作成する。
  `GITHUB_TOKEN` で作成した PR の CI は承認待ち状態になるため、write 権限者が Actions タブから
  承認して実行する必要がある。

ワークフローの外部依存 (`uses:` / `container:`) は commit SHA またはイメージ digest で固定し、
バージョンをコメントで併記する。

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

<!-- mise-tasks -->
## `docs`

- **Usage**: `docs`

Sync the task list embedded in README.md with mise.toml
<!-- /mise-tasks -->
