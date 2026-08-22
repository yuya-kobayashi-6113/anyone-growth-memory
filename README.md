# anyone-growth-memory

Growth案件（ID基盤 + Growth見積・帳票）で、Claude が作業に使う**記憶（判断の蓄積）の正本**を置くリポジトリです。

## なぜここに置くか

記憶は、複数のコードリポジトリ（`anyone-user-platform` と `anyone-growth`）にまたがる判断を含みます。用語の統一、環境の癖、過去に踏んだ落とし穴、次にやること。どちらのコードリポジトリにも属さないため、専用の置き場を用意しています。

Claude Code（Mac）からも、Cowork（クラウド）からも、**同じ1つの正本を読みます。**

**案件ごとに1リポジトリです。** 他の案件の記憶は、それぞれ別のリポジトリにあります。

## 構成

```
anyone-growth-memory/
├── README.md
├── CLAUDE.md               索引。毎回読まれる
└── memory/                 トピック別。必要になった時点で読む
    ├── team.md             体制と前提
    ├── terminology.md      用語統一
    ├── id-platform.md      ID基盤の設計・状況
    ├── growth.md           Growth見積の設計・状況
    ├── growth-documents.md Growth帳票出力の設計・状況
    ├── repos.md            GitHubリポジトリ
    ├── tech-stack.md       技術スタック
    └── verification-env.md クラウド検証環境
```

## 読ませ方

### Claude Code（Mac）

このリポジトリを追加ディレクトリとして渡します。

```
claude --add-dir /path/to/anyone-growth-memory
```

索引（`CLAUDE.md`）まで読ませるには、環境変数を立てます。

```
export CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1
```

### Cowork（クラウド）

GitHub 経由でこのリポジトリのファイルを読みます。まず `CLAUDE.md`（索引）を読み、そこから必要なトピックファイルを読みます。

## 更新の作法

- **索引（`CLAUDE.md`）は薄く保つ。** 詳細は `memory/` 側へ置きます
- 記憶を更新したら、**同じ作業の中でコミットまで済ませます**
- **PR番号・テスト件数・オープンPRの有無は書きません。** 陳腐化するので、数え方だけを書いて数え直します

## ここに置かないもの

- **設計文書と、セッションごとの計画・記録** — Cowork のプロジェクトドキュメント（`design-docs/` と `session-docs/`）に残しています
- **進め方の共通ルール** — `allsmile-ops` プラグインが正です
