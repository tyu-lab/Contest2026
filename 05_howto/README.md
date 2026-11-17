# 操作手順書

本ディレクトリには、コンテストで使用する各種操作手順書を掲載しています。

目的に応じて、以下の手順書を参照してください。
初めて利用する場合は、末尾の[実施者について](#実施者について)で、初回操作の順序と実施者を確認してください。

| 手順書 | 内容 |
|---|---|
| [setup-manual.md](./setup-manual.md) | コンテスト環境、AWS、GitLab、CI/CD等の利用・変更方法を確認するための手順書。付録GにはDB初期化に関する説明・注意事項も掲載しています。 |
| [rds-connection-manual.md](./rds-connection-manual.md) | CloudShell VPC環境からRDSへ接続し、SELECTで疎通確認する手順 |
| [stop-instances-manual.md](./stop-instances-manual.md) | GitLab・CI/CD用インスタンスを停止・再開するための手順書 |
| [user-manual.md](./user-manual.md) | JISA Eatsの消費者、店舗従業員、配達員として画面を操作するための手順書 |
| [staff-system user-manual.md](./staff-system%20user-manual.md) | 運営システムで順位、スコア、利益推移、採点履歴等を確認するための手順書 |
| [traffic-db-reset-manual.md](./traffic-db-reset-manual.md) | 疑似トラフィック後にDBを初期化する際の事前確認・実行・結果確認・異常時の対応 |
| [data_analysis/data_analysis.md](./data_analysis/data_analysis.md) | SageMakerを利用した配達時間予測モデルの確認・改善・デプロイ手順 |

---

## 目的別の参照先

| やりたいこと | 参照先 |
|---|---|
| AWS環境やGitLabを確認したい | [setup-manual.md](./setup-manual.md) |
| アプリケーションを変更・デプロイしたい | [setup-manual.md](./setup-manual.md) |
| RDSへ接続したい | [rds-connection-manual.md](./rds-connection-manual.md) |
| GitLab・CI/CD用インスタンスを停止・再開したい | [stop-instances-manual.md](./stop-instances-manual.md) |
| JISA Eatsの画面を操作したい | [user-manual.md](./user-manual.md) |
| 順位やスコアを確認したい | [staff-system user-manual.md](./staff-system%20user-manual.md) |
| システム状況採点を実行したい | [staff-system user-manual.md](./staff-system%20user-manual.md) |
| 疑似トラフィック後のDBデータを初期化したい | [traffic-db-reset-manual.md](./traffic-db-reset-manual.md) |
| 配達時間予測モデルを確認・改善したい | [data_analysis/data_analysis.md](./data_analysis/data_analysis.md) |

## 実施者について

手順によって、チーム全員が各自で実施する作業と、チームの代表者が実施する作業があります。

AWS環境とGitLabの初回操作は、以下の1〜3の順に進めます。各操作の前提条件・注意事項・完了確認は、リンク先の手順書を参照してください。

| 順序 | 作業 | 実施者 | 参照先 |
|---|---|---|---|
| 1 | AWSマネジメントコンソールへの初回ログイン・MFA設定・再ログイン | チーム全員（各メンバーが自身のアカウントで実施） | [環境設定マニュアル 第1章](./setup-manual.md#1-aws環境アクセス)・[第2章](./setup-manual.md#2-mfa設定) |
| 2 | GitLabへのログイン・対象プロジェクトへのアクセス確認 | チーム全員 | [環境設定マニュアル 第3章](./setup-manual.md#3-gitlabへのアクセス確認) |
| 3 | 初回のCI/CDパイプライン実行 | チーム代表者1名（重複実行を避ける） | [環境設定マニュアル 第4章](./setup-manual.md#4-cicdによるアプリケーションのデプロイ)（実施者の指定は4.5） |

初回のCI/CD確認は、環境設定マニュアル第4章に沿って、実行結果の確認からデプロイ結果の確認まで進めてください。コードを変更する際の操作は、同マニュアルの[付録E. コードの修正方法](./setup-manual.md#付録e-コードの修正方法)を参照してください。

順位やスコアの確認に使用する運営システムの初回ログイン・初期パスワード設定は、上記とは別に、**チーム代表者1名（現地参加メンバー）**が実施します。[運営システム ユーザー操作手順書 3.1](./staff-system%20user-manual.md#31-ログインする)を参照してください。
