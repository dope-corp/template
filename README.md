# template

dope-corp organization の新規リポジトリ作成時に使用する共通テンプレートです。

新規リポジトリを作成する際、「Repository template」としてこのリポジトリを選択すると、以下のファイルが引き継がれます。

- `.gitignore`
- `.github/ISSUE_TEMPLATE/bug_report.md`
- `.github/ISSUE_TEMPLATE/feature_request.md`
- `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/workflows/prek.yml`
- `.github/workflows/gitleaks.yml`
- `mise.toml` / `mise.lock`
- `.pre-commit-config.yaml`
- `renovate.json`

各リポジトリの事情に合わせて、生成後に内容を編集してください。

`.gitignore` はホワイトリスト方式 (`*` で全て無視し、追跡させたいパスだけを `!` で許可) です。
ファイルを追加した際は、対象パスを個別に `!` で許可してください。

`renovate.json` は Renovate の設定です。有効にするには、生成後のリポジトリに対して
organization 管理者が [Renovate GitHub App](https://github.com/apps/renovate) (Mend ホストの
SaaS 版) を個別にインストールする必要があります (このテンプレート自体には Bot の有効化範囲は
含まれません)。ワークフローの外部依存は `uses:` / `container:` フィールドで固定してください
(`run:` 内のシェルコマンドは更新対象外)。

`helpers:pinGitHubActionDigests` による GitHub Actions の SHA 更新は Mend ホスト版でも動作します。
一方 `mise.toml` / `mise.lock` (prek, actionlint) の自動更新は **保証されません**。
`mise.lock` の再生成 (`mise lock` の実行) は Renovate 上で `allowedUnsafeExecutions` に
`mise` を含める必要がある unsafe execution 扱いで、これは self-hosted 限定の global 設定です。
Mend ホスト版でこの許可がされているかは非公開のため、mise ツールのバージョン追従は
Renovate に頼らず、`mise install` / `mise up` を手動またはスケジュール実行で行ってください。

## セットアップ

このリポジトリは [mise](https://mise.jdx.dev/) の利用を前提としています。

```sh
mise trust    # 初回のみ: このディレクトリの mise.toml を信頼する
mise install  # tools (prek 等) をインストールし、pre-commit hook をセットアップする
```

`mise install` を実行すると `[hooks] postinstall` により `prek install` が自動実行され、
`.git/hooks/pre-commit` がセットアップされる。

## CI

- `.github/workflows/prek.yml`: push / pull request 時に、`mise.lock` 通りのツールで
  `.pre-commit-config.yaml` の全フックを `prek run --all-files` で実行する。
- `.github/workflows/gitleaks.yml`: push / pull request 時に、コミット履歴全体を対象に
  [gitleaks](https://github.com/gitleaks/gitleaks) でシークレットスキャンを行う。

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
