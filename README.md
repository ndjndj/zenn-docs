# zenn-docs

## リンク

- GitHub リポジトリとの連携方法について
  - [アカウントにGitHubリポジトリを連携してZennのコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github)

- Zenn CLI のインストール方法
  - [Zenn CLIをインストールする](https://zenn.dev/zenn/articles/install-zenn-cli)

- Zenn CLI の使い方
  - [Zenn CLIで記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)

- 既存記事の連携について
  - [Zennで途中からGitHubリポジトリ連携をはじめるときの手順](https://zenn.dev/zenn/articles/setup-zenn-github-with-export)

## Zenn CLI の使い方

- ファイルの配置について
  - .md で管理
  - 記事は /content/articles に作成
- 記事の作成
  - /content で以下のコマンドを実行
  - 拡張子を除いたファイル名はそのまま slug として機能する
  ```bash
    npx zenn new:article
  ```
- 記事のプレビュー
  - /content で以下のコマンドを実行
  ```bash
    npx zenn preview
  ```