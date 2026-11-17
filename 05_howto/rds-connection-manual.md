# RDS接続手順（参加者向け）

提供された権限・構成の範囲内で、CloudShell VPC環境からRDS PostgreSQLへ接続する手順です。VPC・Subnet・Security Group（SG）のIDやRDSの接続先はチームごとに異なるため、以下のコマンドで自分の環境の値を確認してください。

## 1. 通常CloudShellを開く

環境設定マニュアルのAWSログイン・MFA設定を完了し、対象RDSと同じリージョンで、AWSコンソール上部のCloudShellアイコンから既存の通常CloudShellを開いてください。最初はAWS環境情報の確認に使います。

## 2. RDS・Secret・VPC・Subnet・SGの確認

以下を通常CloudShellで順に実行してください。最初のコマンドの`Account`・`Arn`が自分のチームのAWSアカウント・IAMユーザーを示しているか確認し、一覧から対象RDSの`DBInstanceIdentifier`を`DB_ID`に設定してください。

```bash
aws sts get-caller-identity

aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,Endpoint.Address,DBSubnetGroup.VpcId]' \
  --output table

DB_ID='対象RDS名' # 一覧から対象のDBInstanceIdentifierを入力
DB_INFO=$(aws rds describe-db-instances \
  --db-instance-identifier "$DB_ID" \
  --query 'DBInstances[0].{
    Host:Endpoint.Address,Port:Endpoint.Port,
    User:MasterUsername,Database:DBName,
    Public:PubliclyAccessible,VPC:DBSubnetGroup.VpcId,
    Subnets:DBSubnetGroup.Subnets[*].SubnetIdentifier,
    SecurityGroups:VpcSecurityGroups[*].VpcSecurityGroupId
  }' --output json)
printf '%s\n' "$DB_INFO"

VPC_ID=$(printf '%s' "$DB_INFO" | jq -r '.VPC')
aws ec2 describe-vpcs --vpc-ids "$VPC_ID" \
  --query 'Vpcs[*].{VPC:VpcId,CIDR:CidrBlock}' --output table

# 対象RDSのSubnet（AZ・CIDR）を確認
mapfile -t DB_SUBNET_IDS < <(printf '%s' "$DB_INFO" | jq -r '.Subnets[]')
aws ec2 describe-subnets --subnet-ids "${DB_SUBNET_IDS[@]}" \
  --query 'Subnets[*].{Subnet:SubnetId,VPC:VpcId,AZ:AvailabilityZone,CIDR:CidrBlock}' \
  --output table

# 同じVPCの既存SG候補と、受信・送信の通信許可条件を確認
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'SecurityGroups[*].{
    ID:GroupId,Name:GroupName,VPC:VpcId,
    Inbound:IpPermissions,Outbound:IpPermissionsEgress
  }' --output json

# シークレット名のみ確認（値は出力しない）
aws secretsmanager list-secrets \
  --query 'SecretList[*].Name' --output table
```

AWSコンソールで対象RDSの「接続とセキュリティ」も開き、エンドポイント・ポート・接続案内を確認してください。CLIの`User`はマスターユーザー名、`Database`はDB名です。`Database`が`null`の場合は、同画面の接続案内や既存CloudFormationスタックの出力`DbName`でDB名を確認してください。

Secrets Managerでは`eats-dev-db-password`が対象DBのシークレットか確認してください。見つからない場合や参照できない場合は運営へお問い合わせください。新しいシークレットの作成やパスワードの変更は不要です。

SG一覧のうち、`DB_INFO`の`SecurityGroups`にあるIDの`Inbound`がRDSの受信ルールです。DBポートのTCP受信を許可する送信元CIDRまたはSGを確認してください。CloudShellで選ぶ既存SGの`Outbound`でも、対象RDSへのTCP通信を許可している必要があります。CIDR指定の場合は選ぶSubnetからの通信が許可範囲に含まれるか、SG指定の場合は選ぶSGが許可元に含まれるかを確認してください。`IpProtocol: -1`は全プロトコルの許可です。同じVPCのSGというだけでは接続できるとは限りません。

## 3. 別のCloudShell VPC環境が必要な理由

非公開RDSへは、AWSのリソース情報を確認できる通常CloudShellから、そのままDB接続できるとは限りません。通常CloudShellと、参加者のVPC内で通信するCloudShell VPC環境は別の環境です。対象RDSへ到達するため、RDSと同じVPC・RDSのSubnetに、通信可能な既存SGを関連付けたCloudShell VPC環境を別途作成します。

参考：[AWS公式：CloudShellのVPC環境](https://docs.aws.amazon.com/cloudshell/latest/userguide/using-cshell-in-vpc.html)

## 4. CloudShell VPC環境の作成

1. CloudShell画面の「+」から「Create VPC environment」を選び、環境名を入力してください。
2. 手順2の`VPC`を選び、`Subnets`に表示された既存Subnetのいずれかを選んでください。
3. 手順2でDBへの通信許可を確認した、同じVPCの既存SGを選んでください。RDSに付いているSGを選ぶだけでは、そのSG自身からの受信が許可されるとは限りません。
4. 「Create」を選び、新しいCloudShell VPC環境の起動を待ってください。「Actions」→「View details」で選んだVPC・Subnet・SGを確認できます。

VPC環境を作成できない場合や、通信可能な既存SGが見つからない場合は運営へお問い合わせください。IAM・SGルールを変更して接続しようとしないでください。

参考：[AWS公式：CloudShell VPC環境の作成](https://docs.aws.amazon.com/cloudshell/latest/userguide/creating-vpc-environment.html)

## 5. Secretを使用してpsqlで接続

新しく作成したCloudShell VPC環境で`command -v psql`を実行し、`psql`が使えることを確認してください。見つからない場合は運営へお問い合わせください。

通常CloudShellの変数は新しい環境へ引き継がれません。以下の`DB_ID`を対象RDS名に置き換え、接続情報を再取得してください。コマンドが失敗した場合は、接続へ進まず、秘密情報を除いたエラー文言を運営へお知らせください。

```bash
# 通常CloudShellの変数は引き継がれないため、対象RDS名を再入力
DB_ID='対象RDS名'
DB_INFO=$(aws rds describe-db-instances \
  --db-instance-identifier "$DB_ID" \
  --query 'DBInstances[0].{Host:Endpoint.Address,Port:Endpoint.Port,User:MasterUsername,Database:DBName}' \
  --output json)
DB_HOST=$(printf '%s' "$DB_INFO" | jq -r '.Host')
DB_PORT=$(printf '%s' "$DB_INFO" | jq -r '.Port')
DB_USER=$(printf '%s' "$DB_INFO" | jq -r '.User')
DB_NAME=$(printf '%s' "$DB_INFO" | jq -r '.Database')
```

`Database`が`null`の場合は、RDSの「接続とセキュリティ」の接続案内または既存CloudFormationスタックの出力`DbName`でDB名を確認し、`DB_NAME='確認済みのDB名'`を実行してください。その後、下の接続用コードを実行してください。DB名を設定した後に上のコードを再実行すると、`DB_NAME`が取得結果で上書きされます。

```bash
# シークレットを画面へ出さず、接続するpsqlプロセスだけに渡す
set +x
set -o pipefail
if [[ -z "$DB_HOST" || "$DB_HOST" == null ||
      -z "$DB_USER" || "$DB_USER" == null ||
      ! "$DB_PORT" =~ ^[0-9]+$ ]]; then
  printf '%s\n' '接続情報を取得できませんでした。運営へお問い合わせください。'
elif [[ -z "$DB_NAME" || "$DB_NAME" == null ]]; then
  printf '%s\n' 'DB名を確認してDB_NAMEへ設定し、この接続用コードだけを再実行してください。'
elif DB_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id 'eats-dev-db-password' --query SecretString --output text \
  | jq -er '.password | select(type == "string" and length > 0)'); then
  PGPASSWORD="$DB_PASSWORD" psql -X \
    "host=$DB_HOST port=$DB_PORT dbname=$DB_NAME user=$DB_USER sslmode=require connect_timeout=15"
  unset DB_PASSWORD
else
  unset DB_PASSWORD
  printf '%s\n' 'シークレットを取得できませんでした。運営へお問い合わせください。'
fi
```

シークレットのJSONにある`password`を、画面へ出さずに接続時の`PGPASSWORD`へ渡します。`psql`終了後は変数を削除します。`sslmode=require`はTLS暗号化を指定しています。`verify-full`のように、CA証明書と接続先ホスト名の両方を検証する指定ではありません。`PGPASSWORD`はプロセスの環境変数であり、OSや権限によっては他のプロセスから参照できます。画面に出力しないことだけで秘密情報の保護を保証するものではありません。DBパスワードはAWSログイン用パスワードとは異なります。パスワード・シークレットの値・アクセスキーをIssue、Teams、ログへ貼り付けないでください。

この手順ではVPC環境からRDS・Secrets ManagerのAPIにもアクセスします。DBへの通信に加え、これらのAPIへの通信経路も必要です。CloudShell VPC環境にはパブリックIPが割り当てられないため、Internet GatewayへのルートだけではAPIの公開エンドポイントへ到達できません。既存のNAT経路や対象サービスのVPCエンドポイントを利用できる構成が必要です。APIやDBへ到達できない場合は、選択した環境・リージョン・VPC・Subnet・SGを確認し、解消しない場合は秘密情報を除いたエラー文言と失敗した手順を運営へお知らせください。SG・IAM・ルート等の設定変更は行わないでください。

参考：[AWS公式：PostgreSQLへの接続](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ConnectToPostgreSQLInstance.html)、[シークレットの取得](https://docs.aws.amazon.com/cli/latest/reference/secretsmanager/get-secret-value.html)

## 6. SELECTで疎通確認

`psql`のプロンプトが表示されたら、接続先のDB名とユーザーを確認してください。

```sql
SELECT current_database(), current_user;
```

2026-10-08に予備参加者環境の参加者IAMユーザーで、この接続経路で接続し、DB`eats`・ユーザー`eatsadmin`の結果を確認しました。各チームの接続先・DB名・ユーザーは、自分の環境の取得結果と照合してください。この疎通確認は、各テーブルの参照権限やSQLメトリクス・パフォーマンス分析画面の利用可否を確認するものではありません。

調査のSELECTは対象・条件・取得件数を絞ってください。参照でもDB負荷が発生します。

採点中に運営が評価用として作成・利用しているデータを手動で変更・削除しないでください。また、運営があらかじめ用意した評価用データを、評価を回避したり評価結果を意図的に変えたりする目的で追加・変更・削除・無効化しないでください。動作確認・検証を目的とするデータ操作については、[レギュレーション第3章](../00_overview/regulations.md#3-評価に使用するデータを意図的に変更しない)の規定に従ってください。

接続を終了するには`\q`を入力してください。

参考：[AWS CLI：RDS情報](https://docs.aws.amazon.com/cli/latest/reference/rds/describe-db-instances.html)、[Subnet情報](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-subnets.html)、[SG情報](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-security-groups.html)
