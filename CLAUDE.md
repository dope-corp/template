# CLAUDE.md

dope-corp organization の共通 template リポジトリ。template から生成したリポジトリでは、
このファイルをそのプロジェクトの内容 (目的・ビルド / テスト手順・固有の規約) に書き換える。

## セットアップと検証

- `mise trust` (新しい clone で 1 回) と `mise install`: ツールのインストールと git hooks (pre-commit / commit-msg) のセットアップ。
- `mise run pre-commit`: 全 hook (dprint フォーマット、jactionlint、zizmor、typos、gitleaks 等) を全追跡ファイルに実行する。CI の `prek` job と同じ検証だが、zizmor と jactionlint の online の検査は CI だけで走る (README「ツール」参照)。
- main を checkout しているときの `git commit` は `no-commit-to-branch` hook が止める。`git merge` が競合なく自動で作る merge commit は止めず、`mise run pre-commit` はこの hook を除く。
- `mise run docs`: `mise.toml` の tasks を変更したあと README のタスク一覧を同期する。

## 規約

- commit message は Conventional Commits (`type(scope): subject`)。commit-msg hook の commitlint が検査する。
- json / yaml / markdown / toml は dprint で整形する。手で整えず `mise run pre-commit` に任せる。
- lint の指摘は原因を直し、設定 (規則・hook の option) を緩めて通さない。誤検知と、外部の仕様のために直せない指摘だけを、理由のコメントを添えて最小の範囲で除外する。
  - typos: `typos.toml` の `extend-identifiers` / `extend-words`。
  - zizmor: 該当行の `# zizmor: ignore[<audit 名>]`。
  - jactionlint: 該当行の上の `# jactionlint ignore=<rule>`。jactionlint は zizmor のコメントも読むので、両方が出す finding は zizmor のコメントだけで除外する。
- workflow で action・`vars`・`secrets`・required check を足したら、`.github/jactionlint.yaml` の `forbidden-uses.allow`・`config-variables`・`config-secrets`・`required-checks` にも足す。
- `.gitignore` はホワイトリスト方式。新しいファイルを追跡するときは対応する `!` の行を追記する。
- ツールは `mise.toml` に exact version で pin し、変更したら `mise lock` で `mise.lock` を追従させる (`locked = true` なので、追従させないと手元でも tool を実行できない)。
- workflow の `uses:` は commit SHA で固定し、バージョンをコメントで併記する。
- コメント・ドキュメントは日本語で書き、技術用語・識別子は原語のまま使う。
