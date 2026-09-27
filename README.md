# template

[![CI](https://github.com/dope-corp/template/actions/workflows/ci.yaml/badge.svg?event=pull_request)](https://github.com/dope-corp/template/actions/workflows/ci.yaml)

dope-corp organization の新規リポジトリ作成時に使用する共通テンプレートです。

新規リポジトリを作成する際、「Repository template」としてこのリポジトリを選択すると、以下のファイルが引き継がれます。

- `.gitignore`
- `.github/CODEOWNERS`
- `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/ISSUE_TEMPLATE/bug_report.yaml` / `feature_request.yaml` (issue forms。org の issue type Bug / Feature を付ける)
- `.github/workflows/ci.yaml` (prek / gitleaks)
- `.github/workflows/claude.yaml` / `claude-sweep.yaml` (Claude Code)
- `mise.toml` / `mise.lock`
- `fnox.toml`
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
mise install  # tools をインストールし、pre-commit hook をセットアップする
```

`mise install` を実行すると `[hooks] postinstall` により `prek install` が自動実行され、
`.git/hooks/pre-commit` がセットアップされる。

## ツール

ツールはすべて `mise.toml` の `[tools]` で exact version に pin し、`mise.lock` で
プラットフォームごとの URL / checksum を固定する。CI (`ci.yaml`) もローカルも同じ `mise.lock` から
ツールを解決するため、同じ検証をローカルで再現できる。

```sh
mise exec -- prek run --all-files          # pre-commit hooks を全ファイルに対して実行する
mise exec -- gitleaks git --redact -v .    # コミット履歴全体のシークレットスキャン
```

- [prek](https://github.com/j178/prek): pre-commit hook の実行基盤。hooks は `.pre-commit-config.yaml` で定義する。
- [dprint](https://dprint.dev/): json / markdown / toml / yaml のフォーマッタ (`dprint-fmt` hook)。
  plugin の WASM URL は `dprint.json` に `url@sha256` の checksum 付きで pin する。
- [actionlint](https://github.com/rhysd/actionlint): workflow の静的検査 (`actionlint-system` hook)。
- [gitleaks](https://github.com/gitleaks/gitleaks): シークレットスキャン (CI で実行)。
- [fnox](https://fnox.jdx.dev/): 暗号化ファイル・パスワードマネージャ・クラウドから secret を読み込み、
  環境変数としてコマンドに渡す secret manager。`fnox.toml` の `[daemon]` は、解決済みの secret を
  メモリにキャッシュする daemon を有効化し、`idle_timeout` (12h) 無操作で終了させる設定。
  provider 側で secret を更新した直後は `fnox daemon clear` でキャッシュを破棄する。

### mise.toml / mise.lock を手で変更するとき

`mise.toml` の `[tools]` を手で変更したら `mise lock` を実行して `mise.lock` を追従させ、
両方を同じ commit に含める。CI の mise-action は `mise.lock` があると `mise install --locked` で
インストールするため、`mise.lock` が古いままだと prek job で失敗する。

## 依存関係の更新 (Renovate)

`renovate.json` で以下を Renovate に任せる。

- `mise.toml` のバージョン bump と、それに伴う `mise.lock` の更新 (同じ PR で行われる)。
- `mise.lock` の週次再解決 (`lockFileMaintenance`)。checksum / URL を最新化し、
  fuzzy 指定のツール (後述の `.node-version` 等) は同一 major 内の最新版に追従する。
- `.pre-commit-config.yaml` の hook `rev` と、`language: node` の hook の `additional_dependencies`。
- workflow の `uses:` の commit SHA (バージョンはコメントで併記し、Renovate が両方を更新する)。
- `dprint.json` の plugin URL と checksum (`customManagers`)。

major 以外の更新は 1 つの PR に集約する。このうち minor / patch は `minimumReleaseAge` (7 日) 経過後、
CI green を条件に自動マージする。major は個別 PR で人手レビューする。

## CI

`.github/workflows/ci.yaml` は pull request 時に以下の job を並列実行する。job 名がそのまま
required status check の名前になる。

- `prek`: `mise.lock` 通りのツールで `.pre-commit-config.yaml` の全 hook を `prek run --all-files` で実行する。
- `gitleaks`: コミット履歴全体を対象にシークレットスキャンを行う。gitleaks のバージョン更新は
  Renovate PR で届き、その PR の CI で新しい検知ルールによる全履歴スキャンが走る。

main への push では実行しない (main は PR 必須で、変更は PR の CI で検証してから merge される)。
同一 ref で新しい run が始まると実行中の古い run はキャンセルされる (`concurrency`)。

build / test を持つリポジトリでは、prek が扱わない `npm audit` / test / build を行う `verify` job を
`ci.yaml` に追加し、ruleset の required checks にも `verify` を加える。参照実装:
[dope-corp/web-app の ci.yaml](https://github.com/dope-corp/web-app/blob/main/.github/workflows/ci.yaml)。

workflow の外部依存 (`uses:`) は commit SHA で固定し、バージョンをコメントで併記する。

## Claude Code

### claude.yaml

issue / PR コメント等の `@claude` メンションで
[claude-code-action](https://github.com/anthropics/claude-code-action) を起動する。
runner を起動する前の一次フィルタとして、`author_association` が OWNER / MEMBER のメンションだけを
通す (書き込み権限の厳密な検証は action 内部で行われる)。

### claude-sweep.yaml

毎週月曜に、前回レビュー済み地点 (tag `claude-reviewed`) から HEAD までの差分を
correctness / security / simplification の観点でレビューし、新規の指摘を `claude-sweep` label 付きの
issue として起票する。修正 PR は作らない。

- tag が無い初回は tag を張るだけで終了し (bootstrap)、差分が無ければ Claude を起動しない。
- Claude を動かす job には `github.token` (`contents: read` / `issues: write`) だけを渡し、
  tag の更新は Claude を通らない別 job で行う。
- レビューが失敗した場合は tag を進めず、次回同じ範囲を再レビューする。
- `workflow_dispatch` で手動実行できる。

### secret

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
- 同一 major 内の minor / patch は Renovate の `lockFileMaintenance` が `mise.lock` の再解決で追従し、
  major 更新の提案は Renovate (nodenv manager、デフォルトで有効) が行う。
- nodenv manager は `mise.lock` を更新しないため、`.node-version` の major 更新 PR では
  そのブランチで `mise lock` を実行して `mise.lock` を commit する
  (旧 major の lock のままだと CI の `mise install --locked` が失敗する)。
- `.gitignore` (ホワイトリスト方式) に `!/.node-version` を追記する。
- `verify` job と、eslint / typecheck / commitlint / dprint (language: node +
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
- org の issue types: Bug / Feature / Task。issue form の `type:` から参照する。

### リポジトリ生成後に初回に行う設定

2026-07 に全リポジトリへ統一適用した設定。テンプレートからは引き継がれないため、生成のたびに実行する。

```sh
REPO=dope-corp/<repo>

# merge 方式: squash のみ / squash タイトルは COMMIT_OR_PR_TITLE /
# merge 後にブランチ自動削除 / wiki off / auto-merge 有効化
# (auto-merge は organization レベルのトグルが無く、リポジトリ単位の設定のため必須)
gh api -X PATCH "repos/$REPO" --input - <<'JSON'
{
  "allow_auto_merge": true,
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
# required checks は ci.yaml の job 名に合わせる: `verify` job を追加したら
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

- ディレクトリ単位の環境変数は `mise.toml` に `[env]` を追加して定義するか、`.env` ファイルを
  `_.file = ".env"` で読み込む。ローカルだけの値は `mise.local.toml` (git 管理外) に置いて上書きする。
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

- **Usage:** `docs`

Sync the task list embedded in README.md with mise.toml
<!-- /mise-tasks -->
<!-- dprint-ignore-end -->
