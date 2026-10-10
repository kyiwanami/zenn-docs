---
title: "6年目エンジニアが、さすがにWebサーバー構成をゼロから組んだ"
emoji: "🧱"
type: "tech"
topics:
  - aws
  - cdk
  - cloudfront
  - ecs
  - rds
published: true
published_at: 2026-06-21
---
## いまWebサーバー構成をゼロから組んだ理由

こんにちは。岩波と申します。

新卒で入社して6年目になりますが、自分でWebサーバー構成を組んだことがありませんでした。

最初の4年半は、[ECS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)上のSpring Bootアプリ、Lambda、Step Functions、SQSといったAWSマネージドサービス上のワークロード設計と実装を続けてきました。

その後、部署を異動し、2年ほどAWS AmplifyでWebサービスを扱うようになりました。主にCognito、AppSync、Lambda、DynamoDBを組み合わせています。

最近、Amplifyのようなサーバーレスが登場した現在も、業界ではWebサーバー型が主流だと気づきました。AWSの資格試験でも、サーバー型のケースを多く見かけます。

一方、自分が触ってきたのはアプリ層とAWSマネージドサービス上のワークロードだけでした。その下のVPC、Subnet、Security Group、ALB、RDSは、別チームが整えた状態で渡されていました。

そこで、さすがにAWSの主要技術を触っていないのはまずいと感じ、自分用にWebサーバー構成を一度ゼロから組むことにしました。

VPCを3層に分け、Security Groupのingressを連鎖させ、CloudFrontからALB、ECS、RDSまで経路を通すところまで、自分で書きました。AmplifyやAWSマネージドサービス、別チームに任せていた範囲を、自分で組み立てられるようになりました。

## 組んだもの

![申請一覧画面](/images/web-server-architecture/requests.png)
![資産一覧画面](/images/web-server-architecture/assets.png)
![ユーザー一覧画面](/images/web-server-architecture/users.png)

申請、資産、会社、部署、ユーザーを管理するマルチテナント業務管理WebアプリのMVPを、Spring BootとThymeleafで作りました。UIは適当です。データはもちろんダミーです。

主な機能は次のとおりです。

- マルチテナント分離（会社境界、`company_id`で他社データの参照と更新を遮断）
- CognitoのApp Client分離（運営側用と企業テナント側用）
- ロール認可（運営側のWorkOps管理者、企業テナント側の管理者・編集者・閲覧者）
- 申請管理（下書き、提出、承認、却下、差戻し）
- 資産管理と会社別マスタ管理

ここからは、このアプリを動かすAWSのWebサーバー構成と、組む途中でハマったことを書きます。

## リクエスト経路で見る全体構成

```mermaid
flowchart LR
  user["ユーザー"] --> cf["CloudFront"]
  cf -->|"VPC origin"| alb["内部 ALB"]
  alb --> ecs["ECS Fargate"]
  ecs --> rds[("RDS MySQL")]
  ecs -. "認証" .-> cognito["Cognito"]
  ecs -. "シークレット読み出し" .-> secrets["Secrets Manager"]
  ecs -. "設定読み出し" .-> ssm["SSM Parameter Store"]
  ecs -. "ログ書き込み" .-> logs["CloudWatch Logs"]
  cognito -. "Hosted UI コールバック" .-> cf
  edge["EdgeStack の Custom Resource Lambda"]
  edge -. "deploy 時に callback URL 書き戻し" .-> cognito
  oidc["GitHub OIDC Provider"]
  ecr["ECR"]
  gha["GitHub Actions"] --> oidc
  oidc --> ecr
  ecr --> ecs
```

ユーザーのリクエストは、CloudFront → 内部ALB → ECS Fargate → RDS MySQLの順に進みます。

認証や設定、ログ、デプロイには、[Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html)、[Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)、[SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)、[CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)、[ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html)とGitHub Actionsを使っています。

CloudFrontの後ろにネットワーク、コンテナ、リレーショナルデータベースを配置した構成です。特殊な要素はありませんが、自分でゼロから組んだのは今回が初めてです。

## CloudFrontからRDSまで、リクエスト経路を辿る

この章では、ユーザーのリクエストがCloudFrontからRDSまで届く経路を順に追います。

CloudFrontを入口に置いた判断そのものは別の章にまとめたので、ここでは経路の話に絞ります。

ViewerからCloudFrontまではHTTPSです。CloudFrontから内部ALB、内部ALBからECS taskまではHTTPで通信します。

[CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)側の設定はcaching disabled、[origin request policy](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-origin-requests.html)は`ALL_VIEWER_EXCEPT_HOST_HEADER`を選んでいます。

origin request policyは、CloudFrontが受け取ったリクエストのうち、どのヘッダやクエリをoriginへ引き渡すかを決める設定です。

以降、subnetやSGへの内向きの通信許可をingress、外向きの通信をegressと書きます。

VPCを3層のsubnetに分けています。

- public subnet：NAT Gatewayを置いて、app private subnetからのegress（ECR image pull、CloudWatch Logs）を出している
- app private subnet：ECS Fargate taskと内部ALB
- db isolated subnet：RDS MySQL

```mermaid
flowchart TB
  subgraph VPC["VPC"]
    subgraph PublicSubnet["public subnet"]
      NAT["NAT Gateway"]
    end
    subgraph AppPrivate["app private subnet"]
      ALB["内部 ALB"]
      ECS["ECS Fargate task"]
    end
    subgraph DbIsolated["db isolated subnet"]
      RDS[("RDS MySQL")]
    end
  end
  CF["CloudFront managed prefix list"] -->|"443"| ALB
  ALB -->|"8080"| ECS
  ECS -->|"3306"| RDS
  ECS -. "egress" .-> NAT
```

Security GroupのingressはCloudFrontのmanaged prefix list → ALB → ECS task → RDSの順で連鎖させています。

[managed prefix list](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-aws-managed-prefix-lists.html)は、CloudFrontの送信元IPレンジをAWSが管理・更新するリストです。SGの許可元としてそのまま指定できます。

CloudFrontがingressの起点になるとは、それまで一度も意識していませんでした。そのため、ALBのSG inboundに[CloudFrontのmanaged prefix list](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/LocationsOfEdgeServers.html)を設定するところで一度つまずきました。

最初はALBのinboundを設定せずにCloudFrontからリクエストを送ったところ、504が返ってきました。`com.amazonaws.global.cloudfront.origin-facing`という名前でCDK lookupを行い、取得したprefix list IDをinboundに設定したら通信できました。IDを`pl-...`で固定する形にはしていません。

ECS taskのimageはECRからpullします。

ECS taskが必要とする設定値は、性質ごとに置き場所を分けました。`SPRING_PROFILES_ACTIVE`とJDBC URLはSSM Parameter Storeに置いています。DBのusernameとpasswordは、RDSが自動生成したSecrets Managerのシークレットから読み込みます。CognitoのURLとClient IDは、CDK上で環境変数として直接渡しています。

個人で動かす検証環境なので、サイズは最小にしました。

- RDSは`db.t4g.micro`
- ECS task：0.5 vCPU / 1 GB
- ECR image retention：2世代
- CloudWatch Logs retention：7日

これで開発期間2週間のあいだAWS料金は合計5ドル程度に収まりました。

## CDK Stackを、立てっぱなしの側と必要なときだけ立てる側に分ける

5ドルで済んだのは、CDK Stackを「立てっぱなしの側」と「必要なときだけ立てる側」に最初から分けたからです。

アプリを触らない時間帯は、料金がかかる部分だけdestroyで削除できるようにしたかったので、それを基準にStackを分けました。

```mermaid
flowchart TB
  subgraph Keep["立てっぱなしの側（料金ほぼゼロ）"]
    Deploy["DeployStack"]
    Foundation["FoundationStack"]
    Identity["IdentityStack"]
    Config["ConfigStack"]
    Secret["SecretStack"]
    Registry["RegistryStack"]
    Logs["LogsStack"]
  end
  subgraph Destroy["必要なときだけ立てる側（料金が乗る）"]
    Data["DataStack（RDS）"]
    Egress["EgressStack（NAT）"]
    Edge["EdgeStack（CloudFront）"]
    AppRuntime["AppRuntimeStack（ECS と ALB）"]
  end
  Foundation -. "props" .-> Data
  Foundation -. "props" .-> Egress
  Identity -. "props" .-> Edge
  Registry -. "props" .-> AppRuntime
```

立てっぱなしの側はVPC、IAMのOIDC、SSM、Secrets Manager、ECR、CloudWatch Logs、Cognitoです。

これらは存在しているだけでは料金がほぼかかりません。Secrets Managerのシークレット数やECRのイメージ容量に応じた料金は多少かかりますが、ほぼ無視できる額です。

これらを残しておくと、動作確認のサイクルが短くなります。毎回作り直すとOIDC ProviderやCognito User Poolのsubが変わり、設定が合わなくなるため、立てっぱなしの側に置きました。

subはCognitoがユーザーごとに発行する不変のIDです。User Poolを作り直すと、同じユーザーでも別の値になります。

必要なときだけ立てる側はRDS、NAT、CloudFront、ECS、ALBです。

動作確認したいときだけdeployして、確認が終わったらdestroyする運用にしています。

deployは`app-deploy-dev` workflowで`confirm_runtime_deploy`をtrueにして、手動で起動します。workflowはDataStack、EgressStack、EdgeStack、AppRuntimeStackを順にdeployします。その後、ECS serviceが安定するのを待ち、CloudFront経由で`/actuator/health`が200を返すか確認します。確認は最大10回までリトライします。destroyは手元から逆順に実行します。

RDS（DataStack）を必要なときだけ立てる側に置くかは、最初は迷いました。データを保存するRDSは残したくなりますが、立てっぱなしにすると毎月の料金がかかり続けます。WorkOpsのAWS devでは、Flyway migration V1〜V8と`db/seed/aws-dev`のseedで毎回ゼロから再構築する設計にしました。確認データを永続させるためのDBではないと割り切っています。

[Flyway](https://documentation.red-gate.com/fd)は、SQLファイルに版番号を付け、どこまで適用済みかをDB側に記録するマイグレーションツールです。seedは、動作確認用の初期データを投入するSQLを指しています。

`db/seed/aws-dev`の`V6__insert_users.sql`では、`cognito_sub`をNULLで初期化します。実際のCognito連携時に、WorkOpsの管理導線から`AdminCreateUser`を使って書き戻します。これでDataStackもdestroyの対象にでき、立てっぱなしの側の月額をほぼゼロに保てています。

CDKのentrypointは単一の`bin/cdk.ts`で、すべてのStackを定義しています。GitHub Actionsのworkflowでdeploy対象のStackを明示し、立てっぱなしの側と必要なときだけ立てる側を分けています。

- `infra-dev` workflow：FoundationStack、SecretStack、ConfigStack、IdentityStack、RegistryStack、LogsStackをdeploy（CI後に自動）
- `app-deploy-dev` workflow：DataStack、EgressStack、EdgeStack、AppRuntimeStackをdeploy（手動dispatch）
- GitHub Actions OIDCのRoleを持つDeployStackだけは最初の1回、手元からdeployしています

[OIDC連携](https://docs.github.com/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)は、GitHub ActionsにAWSのアクセスキーを持たせず、workflowの実行時だけ一時的な認証情報を受け取らせる仕組みです。

workflowはAssumeRoleWithWebIdentityでDeployStackのRoleを引き受けてからdeployします。

```mermaid
flowchart TB
  push["git push main"] --> ci["CI workflow"]
  ci -. "workflow_run" .-> WF1
  dispatch["workflow_dispatch（手動）"] --> WF2
  subgraph WF1["infra-dev workflow"]
    Infra["立てっぱなしの側の deploy"]
  end
  subgraph WF2["app-deploy-dev workflow"]
    Runtime["必要なときだけ立てる側の deploy"]
  end
  WF1 -. "AssumeRoleWithWebIdentity" .-> role["DeployStack の Role"]
  WF2 -. "AssumeRoleWithWebIdentity" .-> role
```

SG本体はFoundationStackに置き、ingress ruleは通信要件を持つStackで定義しました。CloudFront prefix listからALB SGへのHTTP 80の許可はEdgeStackで設定します。ALB SGからECS task SGへの8080の許可は、AppRuntimeStackの`CfnSecurityGroupIngress`で設定します。ECS task SGからDB SGへの3306の許可はDataStackで設定します。

Stackの分け方とSGの通信要件を揃えることで、必要なときだけ立てる側をdestroyしたときに、立てっぱなしの側のSGにruntime通信用のinbound ruleが残らずに済みます。

Stackを分けたことで、次の2つの問題にもつまずきました。

### Custom Resource Providerが暗黙に作るLogGroupが残る

CDKで[Custom Resource](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.custom_resources-readme.html)を書くと、内部でLambda Providerが動き、そのLambdaがCloudWatch LogsにLogGroupを作ります。

このLogGroupはCustom ResourceのStackに紐づいて作られるはずなのですが、StackをdestroyしてもLogGroupが消えませんでした。

`RemovalPolicy.DESTROY`を設定しているつもりだったので、削除されると思っていました。

調べると、Providerが暗黙に作るLogGroupには、CDKの`RemovalPolicy`が適用されていないようでした。

LogGroupをProviderと同じStack内で明示的に作り、[Providerに`logGroup`で渡す形](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.custom_resources.Provider.html)に直しました。これでStackのdestroy時にLogGroupも削除されるようになり、不要なLogGroupが増えなくなりました。

LogGroupはProvider用と、Custom Resourceの実体であるLambda用の2つを別々に作り、それぞれのconstructに渡しています。

### `ImportValue`で参照されているStackを削除できない

Stack間で値を渡すと、CDKは標準では[`Fn::ImportValue`](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-importvalue.html)を使います。

参照されている側のStackを削除しようとすると、「他のStackがexportを使っている」というエラーが出てdestroyできません。

必要なときだけ立てる側のStackをdestroyするたびに引っかかりました。

`defaultCrossStackReferences: weak`を設定し、依存する値はpropsで渡すようにしました。CloudFormation側では[`Fn::GetStackOutput`](https://aws.amazon.com/blogs/devops/simplify-cross-account-and-cross-region-stack-output-references-with-aws-cloudformation-and-cdks-new-fngetstackoutput/)で読む形に変えたところ、削除できるようになりました。

ImportValueを使わなくなったことで、参照によってdestroyが制限される問題を解消できました。

```mermaid
flowchart TB
    subgraph before["標準の ImportValue"]
        A1[FoundationStack] -->|"Export を参照"| B1[AppRuntimeStack]
        N1["参照されている側は destroy 不可"]
    end
    subgraph after["weak 参照へ変更後"]
        A2[FoundationStack] -. "props + Fn::GetStackOutput" .-> B2[AppRuntimeStack]
        N2["片方だけ destroy できる"]
    end
```

## 個人検証環境で省いた構成

開発期間2週間でAWS料金が合計5ドル程度に収まったのは、個人検証環境なので、本番なら入れる構成要素を最初から省いたためでもあります。

省いた項目と、本番で使う場合の想定をまとめます。

**RDSのMulti-AZ**は使っていません。DataStackのMySQLはsingleAZ、`db.t4g.micro`、20 GB gp2です。検証データを毎回Flywayで作り直す前提なので、冗長化は省きました。本番ではMulti-AZにするか、Aurora Serverless v2に置き換えます。

**独自ドメインとACM証明書、Route 53**も使っていません。CloudFrontのdefault domainがそのままHTTPSの入口になるので、ドメイン取得、DNS設定、ACM証明書の発行を省きました。

**WAF**はCloudFrontの前段に設定していません。攻撃面を持たない検証環境なので省きました。独自ドメイン化とあわせて、AWS WAFをマネージドルールで紐付けます。

**NAT Gatewayの冗長化**も省きました。EgressStackのNATは1つだけで、public subnet[0]に置いています。app private subnetは2 AZにあるため、片方のAZがダウンするとECRからのイメージ取得が失敗します。本格運用では各AZに1つずつ置きます。

**ALBのHTTPS listener**は使っていません。内部ALBのlistenerはHTTP 80だけです。CloudFront → 内部ALBのVPC originがHTTPなので、ALBに証明書を設定していません。

**ECS ServiceのAuto Scaling**は設定せず、desiredCountを1に固定しています。検証中に同時アクセスが増える状況がないので、1 taskで十分です。

**CloudWatch LogsとECRの保持期間**は短くしています。ログは7日間、ECRのtagged imageは2世代まで保持し、untagged imageは1日で削除しています。

構成の省略とStackの分離を組み合わせたことで、立てっぱなしの側はほぼ料金がかからない構成になりました。

立てっぱなしの側に残るIAM Role、Cognito User Pool、SSM Parameter Store、Secrets Manager、ECR、CloudWatch LogGroupは、いずれもアイドル時の課金がほぼゼロです。

シークレットの数が少なく、ECRのイメージ容量も小さいうえ、検証中のCognitoのMAUもゼロに近いため、立てっぱなしでも毎月の料金はほぼ無視できる額です。

2週間で5ドル程度かかった内訳は、必要なときだけ立てる側をdeployしていた時間にNAT Gateway、RDS、ECS task、ALBにかかった分、と整理できます。

## CloudFrontを入口に置いた理由

入口は、CDNであるCloudFrontに揃えることにしました。業界一般のWebサーバー構成では、CloudFrontを入口に置くのが標準的な並びです。

検証構成では配信の負荷分散やEdgeキャッシュは使っていません。それでも入口を標準的な形に揃えておくと、WAFの追加、Origin Shieldの有効化、別originの追加、Lambda@Edgeの導入といった拡張を選べます。

CloudFrontのdefault domainは、そのままHTTPSの入口になります。`*.cloudfront.net`という形式で、ACM証明書を別途用意する必要はありません。

独自ドメインを取得しない個人検証環境では、このdefault domainを入口にしました。ACM証明書の発行、Route 53の設定、ALBのHTTPS listenerと証明書の差し替えを省けました。

CloudFrontの[VPC origin](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)を使うと、private subnetの内部ALBに直接到達できます。そのため、ALBをapp private subnetに置いたまま、外からのリクエストを受けられます。

ALBのSG inboundにはCloudFront prefix listだけを許可しています。ALBをpublicにした構成と比べて、インターネットに直接さらさずに済みます。

CloudFrontを入口にしたことで、Cognito Hosted UIの戻り先の設定には2か所でつまずきました。

[Hosted UI](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-managed-login.html)は、Cognitoが用意するログイン画面を使える機能です。ログイン後はApp Clientに登録したcallback URLへ戻ります。

### Cognito callback URLとCloudFrontドメインの循環依存

CognitoのApp Clientのcallback URLにCloudFrontのドメインを入れたいのに、そのCloudFront側のdeployにはCognitoのHosted UI URLが要る、という循環依存が発生しました。

CloudFrontのドメインは、Distributionをdeployした時点で確定します。それまではCognitoのcallback URLに設定する文字列が分かりません。

最初は`https://example.com/`のような仮のcallback URLでCognitoをdeployします。その後、CloudFrontのdeployが終わったら、EdgeStackのCustom Resourceで実際のCloudFrontドメインを書き戻します。この2段階に分けて解決しました。

```mermaid
sequenceDiagram
    participant Deploy as deploy 作業
    participant Cognito
    participant CF as CloudFront
    participant CR as Custom Resource Lambda
    Deploy->>Cognito: placeholder の callback URL で先に deploy
    Deploy->>CF: Distribution を deploy（ここでドメイン確定）
    CF->>CR: EdgeStack の deploy 中に起動
    CR->>Cognito: UpdateUserPoolClient で実際の URL を書き戻し
```

書き戻す側はEdgeStack内のCustom Resource Lambdaで、`UpdateUserPoolClient` APIを呼び出します。

運営側用と企業テナント側用の2つのApp Clientに対して、それぞれ`https://<distribution-domain>/login/oauth2/code/platform`、`/login/oauth2/code/tenant`のURLを組み立てて書き込んでいます。運営側用は`/platform`、企業テナント側用は`/tenant`で受ける形です。

Custom ResourceはCreateとUpdateのときだけ書き戻して、Deleteのときは何もしません。

EdgeStackをdestroyするときにcallback URLを仮の値へ戻してしまうと、次にEdgeStackを立て直すまでCognitoのHosted UIが使えない状態で残ります。そのため、Deleteでは最後に書き込んだURLをそのまま残す形にしました。

### Cognitoから戻るとURLがALBの内部DNS名になる

Cognito Hosted UIでログインしたあと、リダイレクトされた先のページから次のページに遷移しようとすると、絶対URLがALBの内部DNS名になっていて、ブラウザから到達できませんでした。

Spring SecurityがOAuth2ログイン後のリダイレクトで返す絶対URLのhostを、ALBの内部DNS名から組み立てていたことが原因でした。RequestのHostヘッダは使われていませんでした。

```mermaid
flowchart LR
    B[ブラウザ] -->|HTTPS| CF[CloudFront]
    CF -->|"HTTP（VPC origin）"| ALB[内部 ALB]
    ALB --> APP[Spring Boot]
    APP -->|"絶対 URL の host が<br>ALB 内部 DNS 名になる"| B
```

ViewerからCloudFrontまではHTTPS、CloudFrontから内部ALBのVPC originまではHTTPのため、`X-Forwarded-Proto`と`X-Forwarded-Host`の扱いが関係します。

Spring Boot側で`server.tomcat.use-relative-redirects=true`を設定し、相対URLを返すようにしたら解決しました。絶対URLは返さない形です。

相対URLならhostを組み立てる必要がないため、Hostヘッダの違いは結果に影響しません。

## その他、組む途中でハマったこと

### private subnetにいるRDSの中身をどう見るか

RDSはdb isolated subnetに置いているため、手元のMySQLクライアントからは直接接続できません。EC2の踏み台やSSM port forwardingなどを調べ、今回はAWS ConsoleのRDS画面にある「[CloudShell VPC environment](https://docs.aws.amazon.com/cloudshell/latest/userguide/creating-vpc-environment.html)で接続」を使うことにしました。

RDS ConsoleからCloudShell VPC environmentを起動すると、VPC内にCloudShell用のENIが作られます。`mysql`クライアントが入った環境から接続し、`USE workops;`まで実行できました。ENIは仮想のネットワークインターフェースです。これを通じて、CloudShellからVPC内のリソースへ到達できます。

DataStackにはCloudShell VPC environment専用のSGを用意し、self-referencingでMySQL 3306の通信を許可したうえで、RDSに割り当てています。確認が終わったらCloudShell VPC environmentは削除します。

```mermaid
flowchart LR
    Console["RDS Console"] --> CS["CloudShell VPC environment"]
    CS -->|"VPC 内に ENI が建つ"| ENI["CloudShell 用 SG"]
    ENI -->|"3306"| RDS[("RDS MySQL")]
```

## これから足したい部分

構成そのもの以外では、次の2つを残しています。

- 本番相当の運用監視（CloudWatch Alarmの整備、SNS通知）
- バックアップとリストア手順の検証

## ソースコード

@[card](https://github.com/kyiwanami/workops)

## 追記

このWebサーバー構成の上に、AWSネイティブのCI/CDパイプラインを組んだ話も書きました。

@[card](https://zenn.dev/kyiwanami/articles/aws-cdk-pipelines-cicd)

## おわりに

これまで触ってこなかった範囲を自分で書いてみたことで、Webサーバー構成を自分で組み立てられるようになりました。

ところで、今回の開発はAIをフル活用しました。コーディングはすべてコーディングエージェントに任せていて、自分では1行も書いていません。

要件、基本設計、ADR、実装計画は、Notion上でNotion AIと相談して固めました。確定した内容をコーディングエージェントに渡し、実装を任せる進め方です。

コードと設定の最終レビューと採否の判断は、自分で行うようにしました。

この開発手法そのものについては、別記事で改めて書こうと思います。
