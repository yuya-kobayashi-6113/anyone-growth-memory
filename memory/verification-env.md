---
name: verification-env
description: クラウド側で全スタックを組み立てて動作確認する手順。Docker起動・遮断の回避経路(Keycloakと.NET)・Playwrightでの通し操作。「テストはローカル実行が唯一の関門」という旧前提を覆した記録
type: reference
---
# クラウド検証環境(2026-08-18 確立)

> **この文書ができたことで、案件の前提が1つ覆った。**
> 旧: 「クラウドではNuGetが遮断されていて事前ビルドできない」「テストの全件確認はローカル実行が唯一の関門」
> 新: **クラウドで両リポジトリのビルド・全テスト・実ブラウザでの通し確認までできる。**

## できること・できないこと

**できる**: 依存サービス(PostgreSQL / Keycloak / Mailpit)の起動、レルム設定の適用、両リポジトリのビルド、**結合テストを含む全テスト**、5プロセスの起動、**Playwrightでの実操作とスクリーンショット取得**。

**できない**: 小林さんの手元固有の設定ずれの検出。**あくまで「同等の環境を組み直したもの」**であり、手元の環境そのものではない。

## 組み立て手順

### 1. ソースを持ってくる
**クラウドからプライベートリポジトリを clone できない**(認証情報が無い)。デスクトップ経由で取り出す。

1. `device_request_folder_access` で `/Users/kobayashiyuuya/vsc/anyone/growth` の許可をもらう(**小林さんはAllowを押すだけ**)
2. `device_bash` で `git archive --format=tar.gz -o <path>/<repo>-src.tar.gz HEAD` を両リポジトリで実行(**追跡ファイルだけなので1MB以下**)
3. `device_stage_files` でクラウドへ運び、展開する

**注意: `device_bash` はネットワークに出られない。**`git pull` は**小林さんの手元でやってもらうしかない**。**アーカイブ取得の前に必ず pull を依頼する**(一度、pull前の古いHEADを持ってきて検証をやり直した)。

### 2. Dockerを起動する
**既定では daemon が止まっている。**`dockerd` を自分で上げる(rootなので可)。

### 3. 遮断されているものを回避する【重要】
| 必要なもの | 既定の入手先 | 状態 | 回避 |
|---|---|---|---|
| Keycloak 26.5.5 | `quay.io` | **遮断** | **Docker Hub の `keycloak/keycloak:26.5.5` を取得し、`quay.io/keycloak/keycloak:26.5.5` にタグを付け替える**(compose を書き換えずに済む) |
| .NET 10 SDK | `dot.net` / `mcr.microsoft.com` / `packages.microsoft.com` | **すべて遮断** | **Docker Hub の `bitnami/dotnet-sdk`(10.0.400)からSDKを丸ごと取り出してPATHに通す** |
| PostgreSQL / Mailpit / keycloak-config-cli | Docker Hub | 到達可 | そのまま |
| NuGet / npmレジストリ | — | **到達可** | そのまま |

### 4. 立ち上げ
compose を上げ、**Keycloakの判定は `.well-known` が200を返すことだけ**を信じる。そのあと `dotnet build` → API 2本、`pnpm install` → 画面2本。

**GrowthのBFFセッション用SQL(`apps/growth-web/database/*.sql`)は手で適用する。**`.env` は `.env.example` からコピー。

### 5. Playwright
**`/opt/pw-browsers` に Chromium が入っている。**`playwright` パッケージを入れれば動く。ヘッドレスでスクリーンショットまで取れる。

## 踏んだ罠

- **ソースを入れ替えたらコンテナを作り直す。**bind mount が削除済みの実体を掴み続け、**Keycloakのテーマが読み込まれなかった**(`Failed to find LOGIN theme`)。`docker compose up -d --force-recreate keycloak` で解決
- **`node_modules` はアーカイブに入らない。**ソースを再展開したら `pnpm install` をやり直す
- **Docker daemon は落ちることがある。**落ちるとコンテナも止まる。長い作業の合間に確認する
- **Keycloakのパスワード設定メールは件名が「アカウントの更新」**(「パスワード」で検索しても見つからない)
- **メールは Mailpit の API(`/api/v1/messages`)から取れる。**本文からリンクを正規表現で抜くのが早い

## 通し確認の手順(2026-08-18 に実際に流したもの)

1. コンソールでサインアップ → Mailpitから確認リンク → メール確認
2. Mailpitから「アカウントの更新」メール → Keycloakの画面でパスワード設定
3. ログイン → 会社作成
4. 利用サービス画面「未申請」→「連携を承認する」→「開通待ち」
5. Growthのポータル「開通待ち」→「Growthの利用を開始する」→「Growth利用中」
6. コンソールの利用サービスが「利用中」になることを確認
7. Growthでログアウト → 中継画面 → ログイン画面(`prompt=login` 付き)
8. コンソールでログアウト → **ここで不具合を検出した**

**中継画面のように一瞬で消えるページは、遷移を止めると空白になる。**サーバーが返すHTMLを取得して `setContent` で描き直すと、静止画として撮れる。

## 実測値(2026-08-18)
- 共通ID基盤: ビルド約35秒 / **テスト716件が約3分20秒**(単体505・結合201・アーキテクチャ10)
- Growth: ビルド約35秒 / `pnpm run ci` 一式が数分(バックエンド162件を含む)
- 環境の組み立て(ソース取得〜5プロセス起動)は初回で1時間弱

## この環境の使いどころ
- **PRを出す前にビルドとテストを通す**(2026-08-18 に運用として決定)
- **画面の通し確認**。CIも単体テストも守れない領域に網をかける
- **E2Eの自動化の土台**。Growthのv1積み残しP1がこれに当たる

## Macローカルの実機確認用アカウント【2026-08-25 作成】

Claudeが実機確認(ブラウザ操作)で使う。**消さずに使い回す。**

- **アカウント**: `claude-check-20260825@example.com` / パスワードは `check-pass-20260825Y`(ローカル専用。パスワード変更機能の通し確認で X→Y へ変更した・2026-08-25。Mailpit宛のダミーアドレスなので本物のメールは届かない)
  - **メールアドレス変更を1件申請済み**(変更先 `claude-check-new-20260825@example.com`・未確定)。確定画面の確認に使った
- **検証用の会社**: 「クロード確認用の非常に長い会社名で…株式会社テスト×9」(オーナー)。**名前が長いのは省略表示の検証のため。改名しない**
- **組織**: 工事部 > (第一工事課・営業部)。営業部はD&D検証で工事部の子へ移動した状態
- **ローカルの起動コマンド(小林さん指定・2026-08-25)。聞かれたらこれをそのまま答える:**

  ```
  cd /Users/kobayashiyuuya/vsc/anyone/growth/id-platform
  bash tools/dev/devenv.sh down
  bash tools/dev/devenv.sh up
  ```

  (down→up の順。両リポジトリのアプリと依存サービスがまとめて上がる)
- メールは Mailpit(`http://localhost:8025/api/v1/search?query=to:<addr>`)から取る
- **ローカルで「サービスへ戻る」を出すには `ServiceCatalog.Services.growth.EntryUrl` が要る**(appsettings.Development.json に追加済み・2026-08-25)

## worktree の変更は localhost:5101 に出ない【2026-08-25 実測】

**`devenv.sh` が起動する 5101 は「本体リポジトリ」を配信する。worktree で直したものは反映されない。**

- 実測: 5101 の next-server の作業ディレクトリは `<本体>/apps/growth-web`。worktree ではなかった
- **この取り違えで一度「スマホ対応が効いていない」と誤判定し、実装側を不当に差し戻した。**
  実装側が「5101は別チェックアウトを見ている」と指摘して発覚した。**前提の指摘は真に受けて確かめること**
- worktree を実機確認する方法は2つ。**Keycloak の redirect_uri が 5101 固定なので、別ポートで画面だけ上げても認証が通らない**
  1. 5101 の dev サーバー自体を worktree のコードに差し替える(認証まで通したいときはこれ)
  2. API だけ別ポート(5102など)で `dotnet run` し、worktree の `.env` の `GROWTH_API_BASE_URL` をそこへ向ける
- `.env` は本体側からコピーが要る。`.next` が古いときは消す

## Growth のテストは環境変数2つが必須【2026-08-25】

```
export GROWTH_TEST_DATABASE='Host=localhost;Port=55432;Database=growth_ci;Username=growth;Password=growth'
export DOTNET_HOSTBUILDER__RELOADCONFIGONCHANGE=false
```

- **前者が無いと PostgreSQL の結合テストが62件黙ってスキップされ、緑に見える。**検証したことにならない
- 後者が無いとホスト起動が5分でタイムアウトする
- 55432 は検証用DBコンテナ `growth-verify-db`(CIと同じ認証)
- **id-platform の Testcontainers 系はこの経路では動かない**(実行させない)

## 画面の描画結果はテストで守れない【2026-08-25 に2回踏んだ】

`apps/growth-web` の vitest は `environment: "node"` で、DOM を組み立てるテストが書けない。
既存のテストは `renderToStaticMarkup` による静的描画の確認まで。**次の型の不具合は自動テストが全部緑でも起きる:**

1. **CSSセレクタが実物のクラス名と噛み合っていない**(`.workspace-desktop-sidebar` と書いたが実物は `growth-sidebar` だった)
2. **クライアント側の状態が絡む描画の固まり**(マウント判定フラグを再マウント時に戻し忘れ、取得結果を捨て続けて「読み込み中」のまま止まった)

**画面に関わる変更は、必ず実ブラウザで描画結果を見る。**

## CI は両リポジトリとも無効化中【2026-08-25 時点】

コミットとプッシュを繰り返す期間の実行枠を空けるため、依頼により手動で停止(`disabled_manually`)。
復旧するときは Growth が `gh workflow enable 335947838`、id-platform が `327068371`。
