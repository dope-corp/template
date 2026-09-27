# CLAUDE.md

dope-corp organization の共通 template リポジトリ。template から生成したリポジトリでは、
このファイルをそのプロジェクトの内容 (目的・ビルド / テスト手順・固有の規約) に書き換える。

## セットアップと検証

- `mise install`: ツールのインストールと git hooks (pre-commit / commit-msg) のセットアップ。
- `mise exec -- prek run --all-files`: 全 hook (dprint フォーマット、actionlint、gitleaks 等) を実行する。CI の `prek` job と同じ検証。
- `mise run docs`: `mise.toml` の tasks を変更したあと README のタスク一覧を同期する。

## 規約

- commit message は Conventional Commits (`type(scope): subject`)。commit-msg hook の commitlint が検査する。
- json / yaml / markdown / toml は dprint で整形する。手で整えず `prek run --all-files` に任せる。
- `.gitignore` はホワイトリスト方式。新しいファイルを追跡するときは対応する `!` の行を追記する。
- ツールは `mise.toml` に exact version で pin し、変更したら `mise lock` で `mise.lock` を追従させる。
- workflow の `uses:` は commit SHA で固定し、バージョンをコメントで併記する。
- コメント・ドキュメントは日本語で書き、技術用語・識別子は原語のまま使う。
