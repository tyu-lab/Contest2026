# 疑似トラフィック後のDB初期化手順（参加者向け）

疑似トラフィック実行後に残ったデータは、DB初期化用Lambdaで初期化できます。
この処理は注文データだけでなく、店舗・商品・ユーザー情報等も初期化します。必要なデータを事前に保全し、採点や通常の操作と重ならないタイミングで実行してください。

## 1. 実行前の確認

1. 初期化の影響範囲を確認し、残す必要があるデータを保全する。
2. チーム内で実行タイミングを確認し、採点や通常の操作と重ならないことを確認する。
3. 環境設定マニュアル第1・2章に従い、自身の参加者用IAMユーザーでAWSマネジメントコンソールにログインする。
4. 操作対象が自チームのAWSアカウントであることを確認し、リージョンを東京（`ap-northeast-1`）にする。
5. AWSマネジメントコンソールからCloudShellを開く。

## 2. Lambdaの実行

| 項目 | 値 |
| --- | --- |
| 関数名 | `jisa2026-lambda-initialize-for-traffic` |
| リージョン | `ap-northeast-1` |
| 応答の保存先 | CloudShell内の `response_traffic.json` |

CloudShellで次のコマンドを実行します。

```bash
aws lambda invoke \
  --function-name jisa2026-lambda-initialize-for-traffic \
  --region ap-northeast-1 \
  --cli-read-timeout 900 \
  --cli-connect-timeout 60 \
  response_traffic.json && cat response_traffic.json
```

初期化対象には注文データ以外の情報も含まれます。疑似トラフィックで作られた注文だけを削除する手順ではありません。

## 3. 実行後の確認

1. コマンドの出力と `response_traffic.json` の応答内容を確認する。コマンドの終了だけで初期化の成功を判断せず、応答内容にエラーがないか確認する。
2. 注文データに加え、店舗・商品・ユーザー情報等のデータの状態を確認する。

CLI側の `StatusCode` と、応答ファイル内の関数の応答に含まれる `statusCode` は別の項目です。CLI側の `StatusCode` が `200` であっても、それだけでDB初期化処理の正常終了を保証するものではありません。`FunctionError` が表示されている場合は、応答内容も確認してください。詳細は [AWS CLIのinvokeリファレンス](https://docs.aws.amazon.com/cli/latest/reference/lambda/invoke.html) を参照してください。

## 4. エラー・異常時の対応

エラーが発生した場合は再実行せず、運営へ連絡してください。
タイムアウト等で結果を確認できない場合や、実行後のデータ状態に異常がある場合も、再実行せず運営へ連絡してください。
