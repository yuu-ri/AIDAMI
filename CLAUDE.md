# Project Instructions

Projectの名前は`AIDAMI`(Artificial Intelligence for Development And Mutual Innovation)です。
現時点ではAIエージェント・オーケストレーション("Intelligent Office" / PVERRループ)の設計ドキュメント、
関連ブログ記事、事業計画書(`docs/business-plan/`)のみが存在し、本repo自体には実装コード
(`src/`等)はまだない。実際のアプリケーション実装(例: ACPLインタプリタ・`chatEngine`等)は
本repoの外(別のコードベース)で進んでおり、本repoはdocs/設計のみを扱う。

本書はAIエージェント(Claude・ChatGPT/Codex等)向けの、本repo共通の作業規則である。
Claude以外のAIエージェントは[`AGENTS.md`](AGENTS.md)から本書を参照する(内容は分けず、本書を正本にする)。

## 位置づけ・境界

- リポジトリはpublic。コミット・PR・Issueの内容は外部に公開される前提で書く。
- 本repoに実装コードは置かない(ドキュメント・設計・コンテンツのみ)。外部コードベースの
  diff・パッチをそのまま本repoにファイルとして置かない(例: ルートの`create_env_from_user.diff`は
  本repoに存在しない`AiLab/`ディレクトリに対するdiffで、本repo上では適用も実行もできない)。
  同様のファイルを新たに追加しない。
- コンテンツは英語・日本語・中国語が混在している(`docs/business-plan/business-plan.md`は英語、
  `business-plan_zh.md`は中国語訳、`docs/Blog/`配下は主に英語だが日本語の記事もある)。
  既存ファイルを編集する際は、そのファイルが書かれている言語を維持する。

## Branching & PRs

- 長期ブランチは`main`のみ(`develop`はレガシーで現在使われていない)。
- 通常の作業は、`main`からfeatureブランチ(短い説明、例: `add-xxx`)を作り、`main`へPRを出す。
- `main`へ直接commit/pushしない。**自分が出したPRのmergeは禁止**(人間がmergeする)。
- Only open a PR when explicitly asked; never merge your own PR.
- PRの説明には実施した検証内容を載せる。検証していない変更(文書だけ等)は、そう明記する。

## Approvals

- GitHub issueの新規作成、リポジトリの作成・公開範囲(public/private)の変更はhumanの許可が必要。
- remote URLにtokenを埋め込まない(`gh auth`を使う)。secret(APIキー・認証情報等)をコミットしない。

## Documentation

- [`docs/README.md`](docs/README.md)がdocsのディレクトリツリー(ドキュメントマップ)。ただし
  これは当初のテンプレートのままで、`src/`・`tests/`・`dev-log/`等、実際には存在しないパスも
  含んでいる。新しいドキュメントを追加したら、このツリー図を実態に合わせて更新する。
- `docs/`配下が設計ドキュメントの正本:
  - [`docs/roadmap.md`](docs/roadmap.md): ロードマップ(現状「To do」のみで未着手)
  - [`docs/requirements/feature-list.md`](docs/requirements/feature-list.md): 機能要件
  - [`docs/design/`](docs/design/): システム設計(下記「既知の課題」の重複に注意)
  - [`docs/business-plan/`](docs/business-plan/): 事業計画書(英語版・中国語版)
  - [`docs/Blog/`](docs/Blog/): AI関連の調査・解説記事
- エージェントは依頼された範囲でdocsを直接修正できる。変更内容・理由を記録する。

## 既知の課題(作業時に踏みやすい地雷)

- `docs/design/IO.md`(1399行)は内部に"Part 1: Architecture" / "Part 2: MVP Design" /
  "Part 3: MVP Implementation"を含み、これは別ファイルの`IO_Architecture.md`・
  `IO_MVP_Design.md`・`IO_MVP_Implementation.md`と内容が重複している。どちらが正本かは
  現状明確化されていない。これらのいずれかを修正する際は、他方も食い違っていないか確認し、
  気づいた重複・矛盾は無断で削除せず、PRで指摘する。
- `docs/business-plan/business-plan.md`(英語)と`business-plan_zh.md`(中国語)は対になった
  ドキュメント。片方だけ更新すると内容が食い違うので、事業計画の内容を変更する場合は
  両方(または少なくとも差分を明記)を更新する。
- ルートの`create_env_from_user.diff`は本repoに属さない外部コードベースへのdiffで、
  本repoでは適用できない。削除や整理を依頼された場合を除き、無視してよい。
- `docs/requirements/feature-list.md`と`docs/logo/logo-usage.md`はファイル権限が`rwxrwx---`
  (owner/groupのみ読み書き可、other不可)になっている環境がある。該当ユーザーでは
  `Permission denied`になるが、これはファイルが存在しないわけではなく権限の問題なので、
  リポジトリ所有者に権限変更を依頼する。
- `docs/README.md`のツリー図と`docs/_config.yml`の`title`/`description`はテンプレートの
  「My Project」のままで、AIDAMI用に更新されていない。
