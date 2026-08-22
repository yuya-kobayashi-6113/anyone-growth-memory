---
name: github-repos
description: GitHubリポジトリ構成(allsmile-corp)とこの案件固有の環境事情。Macローカルのパス、開発用スクリプト devenv.sh、docker composeの手順と落とし穴、CI事情(基盤とGrowthで別)、結合テストの落とし穴、CIでよく落ちる型。進捗の数字は持たず、数え方だけを持つ
type: reference
---
# GitHubリポジトリ(最終更新 2026-08-19)

> 環境共通の罠(クラウドからMacへgit禁止 / pushでの文字化け / `.github/workflows` の403 / `search_code` が0件を返す / CI無反応時の切り分け)は **allsmile-ops の env-pitfalls スキル**が正。ここには**この案件固有の事情だけ**を書く。

## リポジトリとMacローカルのパス

| | GitHub | Macローカル |
|---|---|---|
| **認証サービス(ID基盤・フェーズ1)** | `allsmile-corp/anyone-user-platform` | `/Users/kobayashiyuuya/vsc/anyone/growth/id-platform` |
| **Growthシリーズ(フェーズ2〜)** | `allsmile-corp/anyone-growth` | `/Users/kobayashiyuuya/vsc/anyone/growth/anyone-growth` |

**【小林さんへコマンドを提示するときは、必ず `cd` を含める】**【2026-08-18 指示】
リポジトリ名とフォルダ名が一致していない(GitHubは `anyone-user-platform`、ローカルは `id-platform`)ため、パスを省くと取り違える。**コマンドを1行だけ渡すときも `cd` から書く。**

```bash
cd /Users/kobayashiyuuya/vsc/anyone/growth/id-platform    # ID基盤
cd /Users/kobayashiyuuya/vsc/anyone/growth/anyone-growth  # Growth
```

Growthの状況は→[[growth]]

## 【最優先】ローカル開発は devenv.sh を使う【2026-08-18 追加】

**手順を手で並べない。**基盤リポジトリの `tools/dev/devenv.sh` が、2リポジトリ・5プロセスの更新と起動をまとめて行う。

```bash
cd /Users/kobayashiyuuya/vsc/anyone/growth/id-platform
bash tools/dev/devenv.sh up        # 更新して全部起動(既定)
bash tools/dev/devenv.sh status    # 何が動いているか
bash tools/dev/devenv.sh down      # 全部止める
bash tools/dev/devenv.sh logs id-api
bash tools/dev/devenv.sh doctor    # 起動前の健康診断だけ
```

- **`up --force`** は git pull だけを省いて続行する(強制pullではない)
- Growthのパスは環境変数で上書き可。既定は基盤から見た兄弟フォルダ
- ログとPIDは `.dev-local/`(gitignore済み)
- **`reset-growth-schema` だけが破壊的。**確認を挟む

**このスクリプトが自動でやること(手でやらない)**:
- Keycloakの起動待ち(**判定は `.well-known` が200を返すことだけ**。ログの見た目では判断しない)
- **`infra/keycloak/` に変更があれば `--force-recreate keycloak-config` を自動実行**(→下の「レルム再適用忘れ」)
- Growthの `.env` 作成(**既にあれば上書きしない**)と `apps/growth-web/database/*.sql` の冪等適用
- 7〜8ポートの占有チェック(**占有しているコンテナ名・プロセス名まで出す**)

**既知の弱点(未修正)**:
- **`apps/id-platform-console/next-env.d.ts` で「未コミットの変更あり」のガードが止まる。** Next.js 16 は `next dev` と `next build` で書き出すパスを変える(`.next/types` ↔ `.next/dev/types`)ため、開発している限り必ず汚れる。**当面は `up --force` で回避**
- **手で起動したプロセスは `down` で止められない**(PIDを記録していないため)。ポートを掴んだまま `up` が止まる。`lsof -nP -iTCP:<port> -sTCP:LISTEN -t` で調べて手で kill する

## パッケージマネージャーは pnpm に統一【2026-08-18】

**両リポジトリとも pnpm 10.15.1。**基盤は npm workspaces から移行した。

- 基盤: `pnpm-workspace.yaml` + `packageManager: "pnpm@10.15.1"`。ワークスペースは `apps/id-platform-console`
- Growth: 元から pnpm。ワークスペースは `apps/growth-web`
- **`engines` は node >=22 / pnpm >=10.15.1**

**【重要な教訓】`packageManager` を書いていないと corepack が勝手に書き足す。**
基盤にはこの指定が無く、**corepack が npm を触るたびに `packageManager: "npm@10.8.2..."` を `package.json` へ書き込んでいた**。`git checkout --` で戻しても次のnpmコマンドで復活するため、**ブランチ切り替えが失敗し続ける**という形で表面化した。しかも `engines` は `npm >= 11` を要求しており、**CIも手元も満たしていない要求**だった(EBADENGINEの警告どまりで気づけない)。**新しいリポジトリでは最初から `packageManager` を明示する。**

**Storybook は pnpm の厳密な node_modules でも問題なく動いた**(移行時にいちばん警戒した箇所。`.npmrc` での巻き上げ設定は不要だった)。

## リポジトリの構成【2026-08-17 大きく変わった】

### anyone-user-platform
**C#とTypeScriptが同居する。**画面はRazor Pagesではなく Next.js に全面移行済み。

- `src/IdPlatform.Api` — **唯一のASP.NET Webホスト**。業務API + OpenAPI/Scalar + コンソール向けの非公開セッションAPI
- `src/AuthGateway` — STS/認可
- `src/Modules/{Organizations,Users}` — Application / Contracts / Domain / Infrastructure の各層
- `src/Shared/{AuditLogging,EventContracts,Hosting,Mailing,SharedKernel}`
- **`apps/id-platform-console` — Next.js + React + TypeScript の画面**。`IdPlatform.slnx` には含まれず、**pnpm workspace**として管理
- `tests/{IdPlatform.ArchitectureTests,IdPlatform.IntegrationTests,IdPlatform.UnitTests}` — C#側のみ。フロントのテストは `apps/id-platform-console` 側のVitest
- `docs/adr/` — **ADRの正**。0001〜0016。控えは `repo-docs/adr/`
- `docs/rules/` — **実装コーディングルールの正**(backend / frontend / design-system / security-and-auth / testing-and-ci)
- `docs/integration/` — 外部サービス連携ガイド・NBSS連携
- `infra/keycloak/themes/idplatform/login/` — Keycloakログイン画面のテーマ
- **`tools/dev/devenv.sh`** — 開発環境の更新・起動スクリプト
- **`src/IdPlatform.Web` はもう無い**(2026-08-17に削除。古いメモや手順に出てきたら読み替える)

### anyone-growth
- `AnyoneGrowth.slnx` / `apps/growth-web`(Next.js・BFF兼用) / `services/Growth.Api`(.NET 10) / `tests/Growth.Api.Tests` / `docs/design/` / `docs/rules/`

## 進捗の数字は書かない。数え直す【2026-08-09 方針】
PR番号・テスト件数・オープンPRの有無は**ここに書き写さない**(書いた瞬間から古くなり、合計だけ偶然合っていて長く気づかれない)。代わりに数え方を置く。

| 知りたいこと | 正準の在り処 | 数え方 |
|---|---|---|
| mainの到達点・マージ済みの流れ | GitHub | `list_pull_requests(state=all, sort=updated, direction=desc)`。ローカルなら `git log --oneline main` |
| オープンPRの有無 | GitHub | `list_pull_requests(state=open)` |
| テストの全件数と成否(ID基盤) | ローカル実行 **または クラウド検証環境** | ルートで `dotnet build && dotnet test`(単体+結合の全件)。**CIの結果では代用できない**(結合テストが走らないため)。フロントは `pnpm frontend:check`。→[クラウド検証環境](project_verification_env.md) |
| テストの全件数と成否(Growth) | CIでもローカルでも可 | ルートで `pnpm run ci`(フロント+バックエンドを通しで検証)。**GrowthのCIは実DBを立てて回すのでCIの結果が使える** |
| CIの成否(ID基盤) | PRコメント | マーカー `<!-- ci-result -->` の結果+失敗時ログ抜粋。pushからコメント更新まで3〜4分 |
| CIの成否(Growth) | 検索で判定 | **PRコメントに書く仕組みが無い。**`get_check_runs` と `get_status` はGitHub MCPの権限不足で**403**。`search_pull_requests` に `status:success` / `status:failure` / `status:pending` を投げる。**マージ済みPRが `status:success` で返ることで、この方法が正しく動くのは確認済み**(2026-08-17)。**確定まで40分以上かかることがある** |

**「どこまで実装したか」は意味の単位で [[id-platform-concept]] と [[growth]] に持つ**。番号ではなくフェーズ名で書く。

## main保護の状態【2026-08-04】
- Ruleset `protect-main` 作成済みだが、**GitHub Freeプランのためプライベートリポジトリでは未強制**(GitHub Team加入で自動有効化)
- **当面は運用ルールで代替**: 全変更はPR経由 / CI緑確認後にマージ / mainへ直接コミットしない
- **この案件での実質の運用形**: 「**クラウド検証環境で自分で緑を確認** → PR → CI緑 + 小林さんのローカル確認 → マージ」【2026-08-18 更新】
- **GitHub Team加入の判断はフェーズ2着手前まで** → **フェーズ2は着手済みなので判断が過ぎている**

## 【重要】`.github/workflows/` はクラウドから書き込めない
**403で書き込み不可。読むことはできる。**ワークフローの変更が要る作業(パッケージマネージャーの移行など)は、**差分を提示して小林さんの手元でコミットしてもらう**。作業計画の段階でこの制約を織り込むこと(pnpm移行で実際に必要になった)。

## ポートとプロセス

| | ポート |
|---|---|
| PostgreSQL(compose) | 5432 |
| Keycloak(compose) | 8080 |
| Mailpit 画面 / SMTP(compose) | 8025 / **1025** |
| 基盤API(OpenAPI・Scalarもここ) | 5076 |
| 基盤 管理コンソール | 5091 |
| Growth Web | 5101 |
| Growth API | 5102 |

**基盤のDB**: `Host=localhost;Port=5432;Database=idplatform;Username=idplatform;Password=devpassword`
**GrowthはID基盤と同じPostgreSQLインスタンスを共有し、`growth` スキーマに分離する。**

### 手で起動する場合(devenv.sh を使わないとき)
**5プロセスをそれぞれ別ターミナルで。**片方がフォアグラウンドを占有して2つ目が実行されない事故が実際に起きている。

1. 基盤ルートで `docker compose up -d`
2. 基盤: `dotnet build` → `dotnet run --project src/IdPlatform.Api`
3. 基盤コンソール: `pnpm install` → `pnpm frontend:tokens` → `pnpm frontend:dev`
4. Growth: `pnpm install` → `dotnet run --project services/Growth.Api --urls http://localhost:5102`
5. Growth Web: `apps/growth-web/.env` を用意 → `pnpm frontend:dev`

### Keycloakの起動確認は curl の 200 だけを信じる
1. `docker compose ps -a` — `keycloak-schema` が `exited (0)`、`keycloak` が `running`、`keycloak-config` が `exited (0)`
2. `docker compose logs -f keycloak-config` — **6.xは差分だけ出すので `Create realm` の行は出ないことがある**。出ないこと自体は異常ではない
3. `curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/realms/platform/.well-known/openid-configuration` → **200 が唯一の確実な判定**

### 【重要】レルム設定を変えたら再適用が要る
**`keycloak-config` は一度走って終わるサービス。**`docker compose up -d` だけでは新しい設定が入らない。

```bash
docker compose up -d --force-recreate keycloak-config
```

**これを忘れて数時間を溶かした**(2026-08-18。ログアウトの戻り先を追加したのに反映されず、Keycloakが「無効なリダイレクトURI」を出し続けた)。**devenv.sh はこれを自動化している。**

**実際の登録値を確認するとき**、`kcadm get clients` の一覧は**簡易表現で `attributes` が空に見える**。属性まで見るには単一クライアントを取る。
```
docker compose exec keycloak /opt/keycloak/bin/kcadm.sh get clients -r platform \
  -q clientId=growth-web -q briefRepresentation=false
```

**`post.logout.redirect.uris` に複数の戻り先を書くときの区切りは `##`**(カンマや空白は不可)。

**ログアウトが効いたかは、Keycloakのセッションを直接数えるのがいちばん確実**【2026-08-18 追加】。画面の見た目では判断しない。
```
docker compose exec keycloak /opt/keycloak/bin/kcadm.sh get users/<userId>/sessions -r platform
```

### ログイン確認用ユーザーの作成
管理画面を触らずコマンドで。**手作業でのレルム設定変更は禁止だが、利用者の追加は設定ではないので可。**

```
docker compose exec keycloak /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 --realm master --user admin --password admin
docker compose exec keycloak /opt/keycloak/bin/kcadm.sh create users -r platform \
  -s username=demo@example.jp -s email=demo@example.jp -s emailVerified=true -s enabled=true
docker compose exec keycloak /opt/keycloak/bin/kcadm.sh set-password -r platform \
  --username demo@example.jp --new-password 'DemoPassw0rd!2026'
```

### 過去に踏んだ罠(すべて修正済み。再発したらここを見る)
- **`schema "keycloak" does not exist` で起動し続けられない** — Keycloak(Liquibase)は `KC_DB_SCHEMA` の区画を**自分では作らない**。composeに一度走って終わる `keycloak-schema` サービスを置いて解決。**PostgreSQLの `/docker-entrypoint-initdb.d` は使わない**
- **keycloak-config-cli が `Cannot resolve variable 'env:...'`** — 原因は**YAMLのコメントに書いた変数書式**。変数展開はコメントを含むファイル全体を走査する。**レルムYAMLのコメントにドル記号+丸括弧を書かない**
- **ポート5432が他プロジェクトのコンテナに取られる**。`docker ps` で犯人を特定して停止する
- `keycloak-config` に `restart` を付けない。設定ミスで再起動を繰り返すと**最初の本当の原因がログから押し流される**
- **Keycloakテーマの綴りを間違えると例外にならず、静かに既定テーマへ落ちる**。`docker compose logs keycloak | grep -i theme` で `Failed to find login theme` を確認する

### READMEが古い箇所【2026-08-18 時点。起動手順は修正済み】
- ID基盤 `README.md`: 設計文書の参照先が `../docs/design/`(存在しない)/ 構成説明に旧表記「部署」が残る
- Growth `README.md`: 「フェーズ1の完了後に着手する」(**すでに並行着手済み**)

## CI事情【基盤とGrowthで別。混同しない】

### ID基盤(anyone-user-platform)
- **結合テストはCIでは実行されない(ビルドのみ)** → 実行は**クラウド検証環境か小林さんのローカル**。**何が自動検証され何がされないかの分担はここが正**
- フロントは `pnpm/action-setup` → `setup-node(cache: pnpm)` → `pnpm install --frozen-lockfile` → `pnpm frontend:check`(トークンのずれ検査 / lint / typecheck / Vitest / Next build / **Storybook build**)
- **`dotnet format --verify-no-changes` はCIにしか無い検査**
- **`search_code` はこのリポジトリを索引しておらず常に0件を返す**。横断的な洗い出しは `get_file_contents` でツリーを辿るしかない(**サブエージェントにこれを伝えないと、参照元の見落としをCIで初めて検出することになる。実際に起きた**)
- **Blacksmith が導入されているが、ランナーは専用ラベルで全部オフライン**。CI設定は `runs-on: ubuntu-latest` なので**実際には使われていない**

### Growth(anyone-growth)
- **PostgreSQL 17 のサービスコンテナを立てて実DBに対して検証する。**ID基盤と違い、CIの結果がテストの成否として使える
- 検証は `pnpm run ci` 一本(コメント規約 → デザイントークン → lint → typecheck → test → build → バックエンドbuild → バックエンドtest)。**どれか1つ落ちると全体が落ちる**
- **ブラウザE2Eはワークフローに無い**(未着手・v1積み残しP1)

## ローカル(Mac)側の注意
- **`dotnet build` はDebug、CIはRelease構成**
- レビュー用ブランチの取得: `git fetch origin && git checkout <branch> && git pull`
- **マージ済みブランチに居ると `git pull` が失敗する**。実行依頼のコマンドには必ず `git checkout` を含める
- マージ済みローカルブランチの掃除: `git fetch --prune && git branch --merged main | grep -v '^\*\|main' | xargs -r git branch -d`
- **`launchSettings.json` が環境を Development に固定している。**本番相当の挙動(API仕様を公開しないこと等)を確かめるには `--no-launch-profile` と設定の明示が要る。手順はID基盤のREADMEにある

## 結合テストの落とし穴【重要】
- **テストクラスごとにPostgreSQLコンテナを起動するため、クラスが増えると接続タイムアウト(flaky)が出る**
  - 対処1: `xunit.runner.json` の `maxParallelThreads: 4` / 対処2: **xUnitのコレクション共有にまとめる**。新規テストは**まず相乗りを検討**
- **ID基盤のCIが結合テストを実行しないため、以下はCIの外(クラウド検証環境かローカル)でしか検出できない**:
  - **EF CoreのSQL翻訳**。翻訳できないと実行時例外
  - **マイグレーションが実PostgreSQLに当たるか**、採番列(identity)をEFが正しく扱うか
  - **ページングの正しさ**。⑥-4では作成時刻カーソルの欠落バグを**結合テストだけが検出**した
- **公開IF・ユースケースのコンストラクタを変えたら、テストのスタブと直接 `new` している箇所も同じPRで直す**
- **テストダブルは名前空間ごとに1ファイルへ集約**。命名は `Fake` + 対象名・`internal sealed`
- **HTTP応答をレコード型に読む場合は private ネスト型を避ける**(internalにする)
- **単体テストにEF InMemoryを入れてある**。**分岐と順序の検証にだけ使う**
- **同名の型が2つのアセンブリにあると CS0433**

## ブラウザでしか捕まらない不具合【2026-08-18 追加】
**サーバーの応答が正しくてもブラウザが従わない**類は、単体テストにも結合テストにも映らない。**実ブラウザでの通し操作が唯一の関門。**

- 実例: CSPの `form-action 'self'` が、フォームPOST後の**Keycloakへの遷移**を拒否し、コンソールのログアウトが無言で失敗していた
- **CSPの `form-action` は送信先だけでなく遷移先にも効く。**外部IdPへ飛ぶ画面を触るときは必ず見直す
- ボタンを**リンク(GET)からフォーム(POST)へ変える改修は、CSP見直しとセット**

## CIでよく落ちる型(先回りする。サブエージェントへの指示書に貼る)
- **`TreatWarningsAsErrors` が true。**警告1つで落ちる。フロントの lint も `--max-warnings=0`
- **CS0246**: strongly-typed IDは `IdPlatform.SharedKernel` のusingが要る
- **CS0535 / CS7036**: 公開IF・コンストラクタを変えたら、テストのスタブ・直接newも同時に直す
- **CS0117**: テストから使うメンバーは**public**にする(InternalsVisibleTo未設定)
- **CS8625**: `JwtBearerOptions.MetadataAddress` は非nullable(`Authority` はnullable)
- **CS0433**: 同名の型が2つのアセンブリにある
- **テストメソッド名に `・`(中黒)は使えない**。DisplayNameの文字列にはOK
- **CA1711(型名の予約サフィックス)**: `Permission` / `Collection` / `Attribute` / `Exception` / `Flags` / `Enum` / `Delegate` / `Queue` / `Stack` などで終わらせない
- **CA1002 / CA1819**: publicなAPIで `List<T>`・配列を公開しない→`IReadOnlyList<T>`
- **CA1861**: 引数に定数配列を直接書かない→`private static readonly string[]` に切り出す
- **CA1859**: privateメソッドの戻り値は具象型 / **CA1305**: 数値の文字列化はInvariantCulture / **CA1848**: ILogger直呼び不可→`[LoggerMessage]`
- **CS0618**: Testcontainers 4.x はイメージをコンストラクタ引数で渡す
- **xUnit2029**: `Where`+`Assert.Empty` 不可→`Assert.DoesNotContain`
- **`Dictionary<K,V> d = [];`(コレクション式)は不可** → `new()`
- **ProblemResults に同じエラーコードを2つ登録すると起動時に例外**(FrozenDictionaryのキー重複)

### NuGetの脆弱性監査で restore ごと落ちる
- **新規パッケージは最新安定版を指定する。**古い版の脆弱性監査警告(NU190x)がエラー化する
- **依存で連れてくる推移的パッケージも対象になる。**実例: `Microsoft.AspNetCore.OpenApi` を入れると `Microsoft.OpenApi` の脆弱な版が付いてきて NU1903 で restore が落ちた。**安全な版を直接 `PackageReference` に明示して上書きする**
- **このとき `dotnet format` は「Restore operation failed.」しか出さず、NuGetのエラー本文が見えない。**原因が分からないときの第一容疑にする
