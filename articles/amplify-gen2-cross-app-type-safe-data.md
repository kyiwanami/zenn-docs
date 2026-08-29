---
title: "Amplify Gen2アプリ間で型安全にデータ連携する"
emoji: "🦺"
type: "tech"
topics: ["aws", "amplify", "appsync", "typescript", "codeartifact"]
published: false
publication_name: "trust"
---

## はじめに

こんにちは。
エンジニアの岩波です。

AIエージェントの台頭により、コーディングの多くを任せられるようになりました。
ただし、その前提となるルールやハーネスの整備は、以前より重くなったと感じます。

「いかに判断の余地を減らすか」「いかに安全に開発するか」という仕組みは、依然として人間が設計する必要があります。

データ連携では、方式を選ぶ判断が考慮事項の量を左右し、開発速度と品質にも影響します。
その判断を残すADRも欠かせません。

この記事では、Amplify Gen2のアプリ間データ連携に、手作業を減らしながら型安全に開発できる仕組みを導入した過程を書きます。

![図1：Amplifyアプリ間のデータ授受](/images/amplify-gen2-cross-app-type-safe-data/fig-00-cross-app-connection.png)

## 背景

弊社では、Amplify Gen2を使用して小規模な業務アプリを開発運用しています。

案件によってはアプリごとにリポジトリを分けるマルチリポジトリ方式を採用しており、同一顧客向けの機能であっても、用途ごとに独立したAmplifyアプリとして構成することもあります。

その中で、同一顧客向けアプリ間でデータを連携する必要が生じています。
連携の対象となるアプリは、一部既に本番運用を開始しています。
アプリごとに担当する開発者が異なり、リリース時期も異なります。
連携を追加する場合も、稼働中のアプリの構成は維持する必要があります。

このため、複数のリポジトリに分かれたAmplifyアプリ間で、型安全にデータを連携する仕組みが必要になりました。

:::details 対象

対象は、Amplify Gen2のData機能とAppSyncを使用し、複数のAmplifyアプリ間でマスタデータを共有する開発者です。

Amplify Gen2、AppSync、npmパッケージに関する基本的な知識を前提とします。

記載するのは、現在の構成を採用するまでの経緯、実装時に発生した問題、そして契約パッケージと接続先AppSyncの整合性検査です。

:::

:::details 用語

- **マスタ提供側**：マスタデータを管理し、ほかのアプリへ公開するAmplifyアプリです。
- **マスタ呼び出し側**：マスタ提供側が公開したデータを参照するAmplifyアプリです。

:::

## Amplifyのアプリ間連携に残る手作業

Amplify Gen2には、別のAmplifyアプリのDataを呼び出す[標準手段自体はあります](https://docs.amplify.aws/react/frontend/data/connect-to-API/)が、型生成を含む選択肢は限られています。
たとえば、[AppSync API IDを指定したクライアントコード生成は未対応](https://github.com/aws-amplify/amplify-backend/issues/1903)です。

型安全にするにはCDKなどで独自構築したり、契約ファイルを連携したりする工夫が必要になります。

特に呼び出し側は、ネイティブ機能だけでは連携用のGraphQLと型定義を自分で用意する状態になり、開発体験があまりよくありません。

こういった要件が出た当初は、暫定で型定義ファイルを手渡ししたりGraphQLを自分で書いたりする必要があり、大きな負担でした。

## やりたかったこと

やりたかったのは、アプリをまたいでも単一アプリ内と同じ開発体験を維持することです。

呼び出し方は標準の`client.models.Post.list()`のまま変更せず、接続先のクライアントだけを差し替えます。
このように、同じ形で書くことで負担を減らすことを目指しました。
担当者が違っても、自アプリのコードと連携コードを同じ基準でレビューできます。
連携専用のAPI操作方法と型管理方法を、新たに習得する必要もありません。

コーディングエージェントに対しても、追加の前提説明なしで既存のAmplify Dataの知識がそのまま通用します。
そのため保守性の向上が見込めます。

型によって「何ができて何ができないか」を明示し、開発速度と品質を高めることも狙いました。

## 型安全を成立させる条件

型安全にするには、呼び出し側が参照する型と、実際に稼働しているAppSyncのSchemaが一致している必要があります。

難所は、それをどのように自動化するかでした。
つまり、型の正本をどこに置くか、そして正本と稼働中のAPIのズレをどう検出するか、という二点です。
アプリ間をどう繋ぐかは大きな主題ではありません。
ここはAmplifyの機能では提供されていません。

## 構成の概要

アプリ間連携専用のREST APIは新設せず、呼び出し側から提供側のAppSyncへ直接接続する方式にしました。

提供側の`amplify/data/resource.ts`に定義された全Schema型と、デプロイ済みAppSyncの全model introspectionを、npmの契約パッケージへ収録します。
契約パッケージは[CodeArtifact](https://docs.aws.amazon.com/codeartifact/latest/ug/welcome.html)から配布します。

なぜREST APIを採用しなかったかは後述します。

## 呼び出し側の使用方法

### 呼び出し例

呼び出し側は、次のコードで提供側のマスタデータを参照します。
外部`endpoint`と`authMode`を指定した接続は、[Amplify公式の標準機能](https://docs.amplify.aws/react/frontend/data/connect-to-API/)です。

```typescript:provider-client.ts
import { createProviderClient } from "@your-scope/provider-contract";

const providerClient = createProviderClient({
  endpoint: providerGraphqlEndpoint,
  authMode: "userPool",
});

const { data, errors } = await providerClient.models.Post.list({
  limit: 100,
  selectionSet: ["id", "title", "comments.*"],
});
```

自分自身のデータを操作する場合は、標準通り以下の形です。

```typescript:own-client.ts
import { generateClient } from "aws-amplify/data";
import type { Schema } from "../../../amplify/data/resource";

export const client = generateClient<Schema>();

const { data, errors } = await client.models.OwnData.list({
  limit: 100,
  selectionSet: ["id", "name"],
});
```

このように、クライアントを選択するだけで、そのデータ操作方法は同一にできます。
かつ型も推測できるため安全な開発が可能です。

### 使用できる操作

呼び出し側は、自身のAppSyncと提供側のAppSyncに対して、Amplify Data標準のインターフェースを使用します。

- models、queries、mutations、subscriptions
- `selectionSet`による関連取得
- pagination

呼び出し側では、GraphQLと型定義を手書きしません。
外部連携専用のAPI操作方法と型管理方法を追加で習得する必要もありません。
**自アプリのコードと連携用コードで同じ実装方法を使用するため、担当者が異なる場合も同じ基準で開発、レビューできます**。

## 採用した構成

提供側は、`amplify/data/resource.ts`でのSchema定義、AppSyncへのデプロイ、契約パッケージの公開を行います。
提供側の既存の業務実装は変更しません。

提供側の`Schema`を契約の正本とし、同じSchemaを提供側AppSyncへデプロイします。

ここで使用したのは、AWSの **CodeArtifact** です。
これにより、同一のインターフェースで呼び出すために必要な型とruntime実装をnpmの契約パッケージへ収録して配布することができます。

実データは、呼び出し側から提供側のAppSyncへ直接送信します。
連携用のREST APIは作成せず、REST APIを中間経路としても使用しません。
呼び出し側からDynamoDBを直接参照することもしません。

![図2：全体構成](/images/amplify-gen2-cross-app-type-safe-data/fig-01-overall-structure.png)

### 構成要素

全体は、次の三つの要素で構成します。

1. **契約パッケージ**：npmパッケージとして作成します。提供側の全Schema型を`ProviderSchema`として収録し、デプロイ済みAppSyncのoutputsから取得した全model introspectionを収録します。outputsの`version`は、introspection形式の識別子として保持します。実行時用のmodel introspectionは、以下ではruntime introspectionと表記します。提供側のAppSyncに接続するData clientの生成関数も、このパッケージへ収録します。
2. **CodeArtifact**：契約パッケージを配布するprivate npm registryとして使用します。呼び出し側は契約パッケージのバージョンを完全一致で指定します。
3. **提供側AppSyncへの直接接続**：実データの通信経路として使用します。REST APIを経由せず、提供側の認可とResolverを使用します。

提供側のモデル変更は、契約パッケージの新しいバージョンとして呼び出し側へ伝播します。
呼び出し側が新しい契約パッケージへ更新したとき、互換性のない変更はコンパイルエラーとして検出されます。
つまり、実行時ではなくビルド時に検出でき、CIで止められます。

## 採用までの経緯

解決対象は、提供側のデータモデル変更に伴い、呼び出し側で型定義と呼び出しコードを人手で修正していた問題です。
修正内容はファイルの受け渡しによって共有されていました。
修正時期、修正担当者、対応対象モデルの記録が残りませんでした。

現在の構成へ至るまでに、二つの構成を試行しました。
試行した二つの構成は、いずれも要件を満たしませんでした。

### API GatewayとOpenAPI

まず、提供側の前段にAPI Gatewayを配置し、OpenAPI定義を公開契約とする構成を試行しました。
OpenAPIから呼び出し側の型を生成できますし、REST APIであるため呼び出し側の技術スタックを限定しません。
表面的には要件を満たす構成でした。

ただ、Amplify Gen2のデータモデルと公開契約で、正本が二つに分かれました。
Amplify Gen2のデータモデルでは`resource.ts`が正本ですが、試行した構成ではOpenAPIが公開契約の正本となります。
**内部モデルを変更しても、人がOpenAPIを変更しない限り呼び出し側へ伝播しません。
**モデル変更のたびにOpenAPIを追従させる作業と、**型生成ファイルを呼び出し側へ受け渡す作業が発生**しました。
結局、排除対象だった人手による同期作業が残りました。

REST APIを見送った判断の根拠は、REST APIとOpenAPI定義を提供側で稼働させ、呼び出し側で取り込む運用コストです。
提供側はAPI GatewayとLambdaを新設して稼働を維持し、OpenAPI定義の生成、公開、バージョン管理の経路を用意します。
呼び出し側は、OpenAPIから型を生成する処理を各リポジトリのビルドへ組み込み、維持します。

提供側のAppSyncへ直接接続する構成では、これらのリソースと経路を新設しません。
既存のAppSyncと、そこに定義した認可規則とResolverをそのまま使用します。

### DynamoDBの直接参照

次に、提供側のDynamoDBテーブルを呼び出し側AppSyncのデータソースとして直接接続する構成を試行しました。
中間APIが不要になり、データ経路も最短になりました。

ただ、DynamoDBはキー以外の属性の型を保持しません。
提供側が項目を追加しても、呼び出し側は追加項目の型を自動取得できず、追加項目を目視でSchemaへ追加する作業とResolverの再実装が必要になりました。
提供側がAppSyncに定義した認可規則と業務処理も経由しません。
結局、型伝播と認可に関する問題が残りました。

二つの試行結果から、要件を定義しました。

- 公開契約の正本を提供側のSchema定義そのものとし、提供側のSchema変更を連携用APIで隠さないこと。
- 提供側のモデル変更を契約パッケージの更新として呼び出し側へ伝え、Schema変更を呼び出し側のコンパイル時に型エラーとして検出すること。
- 呼び出し側でGraphQLも型も手書きせず、通常のAmplify Data clientと同じインターフェースで提供側のマスタデータを操作すること。
- 作業者が実行する手順を、依存関係のインストールと通常のビルドに限定すること。
- 呼び出し側が増加しても提供側の作業量が呼び出し側の数に比例しない構成にすること

## 公開契約とData client

### 公開契約の正本は提供側のSchema

公開契約の正本は、提供側の`amplify/data/resource.ts`に定義された`Schema`とします。
提供側は、同じSchemaを使用して自身のAppSyncをデプロイします。

契約パッケージの`ProviderSchema`は、提供側の`Schema`を型としてそのまま再公開します。
契約パッケージには、提供側の全Schema型を含めます。
呼び出し側が実行できるモデルと操作は、提供側のSchemaに定義された認可規則と、リクエストに使用する認証方式によって制御します。

### 契約パッケージの中身

契約パッケージには次の3点を収録します。

1. **`ProviderSchema`**：提供側の`amplify/data/resource.ts`に定義された全Schema型を保持します。
2. **デプロイ済みAppSyncの全model introspection**：提供側の対象環境をデプロイした後、[`ampx generate outputs`](https://docs.amplify.aws/react/reference/cli-commands/)でoutputsを生成します。outputsの[`data.model_introspection`](https://docs.amplify.aws/react/reference/amplify_outputs/)は、モデル単位または操作単位で絞り込まず、そのままruntime introspectionとして使用します。outputsの`version`は、introspection形式の識別子として保持します。
3. **`createProviderClient`**：`ProviderSchema`とruntime introspectionを内包し、提供側のAppSyncに接続するData clientを生成します。

型の正本は、提供側の`resource.ts`に定義された`Schema`です。
runtime introspectionは、デプロイ済みAppSyncのoutputsから生成します。
型と実行時情報の生成元を分けることで、Schemaの型と、実際に稼働しているAPIのruntime情報を一つの契約パッケージへ収録します。

![図3：契約パッケージの中身](/images/amplify-gen2-cross-app-type-safe-data/fig-02-contract-package-contents.png)

### `createProviderClient`

#### 通常のData clientと同じ操作を返す

`createProviderClient`は、提供側のAppSyncへ接続するAmplify Data clientを生成し、呼び出し側には通常のAmplify Data clientと同じインターフェースを返します。

#### 型情報だけでは動作しない理由

Amplifyの`generateClient`へ`ProviderSchema`の型引数と外部`endpoint`を指定すると、型上は`client.models`を使用できます。
この状態では、エディタの補完と型検査が機能します。

:::message alert

ただし、models、enums、queries、mutations、subscriptionsのruntime実装は構築されません。
そのため、型検査に成功しても、`client.models`の呼び出しは実行時に失敗します。
型が通ったので動くはずだ、とはならないわけです。

:::

![図4：型だけの場合とruntime追加後の比較](/images/amplify-gen2-cross-app-type-safe-data/fig-03-types-vs-runtime.png)

#### runtime実装の追加

`createProviderClient`は、次の順序でProvider用Data clientを構築します。

1. `generateClient`で外部endpoint用のクライアントを生成します。
2. 契約パッケージに収録したoutputsを`parseAmplifyConfig`へ渡します。
3. `API.GraphQL`からProvider設定を取得します。
4. Provider設定にmodel introspectionが含まれることを検証します。
5. Provider設定を`addSchemaToClient`へ渡します。
6. models、enums、queries、mutations、subscriptionsのruntime実装をクライアントへ追加します。

```typescript:createProviderClient.ts
// createProviderClientの内部にある主要処理
export function createProviderClient(
  options: CommonPublicClientOptions,
) {
  const client = generateClient<
    ProviderSchema,
    CommonPublicClientOptions
  >(options);

  const providerConfig = parseAmplifyConfig(providerOutputs).API.GraphQL;

  // providerConfigにmodel introspectionが含まれることを検証する
  addSchemaToClient<ProviderSchema>(
    client,
    providerConfig,
    getInternals,
  );

  return client;
}
```

![図5：createProviderClientの処理順序](/images/amplify-gen2-cross-app-type-safe-data/fig-04-create-provider-client-sequence.png)

#### 呼び出し側へ公開するインターフェース

呼び出し側は、`endpoint`と`authMode`を指定して`createProviderClient`を呼び出します。
戻り値は、Amplify標準のData clientです。
`addSchemaToClient`と`getInternals`は呼び出し側へ公開しません。

#### 低レベルruntime APIへの依存

:::message alert

[`addSchemaToClient`](https://github.com/search?q=repo%3Aaws-amplify%2Famplify-js+addSchemaToClient&type=code)と[`getInternals`](https://github.com/search?q=repo%3Aaws-amplify%2Famplify-js+getInternals&type=code)は、Amplifyの公開APIではありません（[公開APIリファレンス](https://aws-amplify.github.io/amplify-js/api/)には含まれません）。
Amplifyの将来のバージョンで変更される可能性があります。

:::

だからこそ、依存箇所は契約パッケージの内部へ隔離します。
Amplifyの更新によって低レベルruntime APIが変更された場合は、契約パッケージの一箇所を修正します。
低レベルruntime APIの変更を、各呼び出し側の実装コードへ波及させません。
Amplifyを更新するときは、契約パッケージと呼び出し側の互換性を再検証します。

### 型と実データの流れ

型とruntime実装の配布経路と、実データの通信経路は分離します。
型とruntime実装は、契約パッケージからCodeArtifactを経由し、`npm ci`によって呼び出し側へ配布します。
実データは、呼び出し側から提供側のAppSyncへ直接接続して送信します。

呼び出し側は、契約パッケージのバージョンを完全一致で指定します。
ビルドに使用した契約は`package.json`とロックファイルに記録されるため、同一コミットから同一の契約で再ビルドできます。
呼び出し側が増加しても、提供側が契約パッケージを公開する作業は一度だけです。

実データについては、提供側に定義した認可規則とResolverをそのまま使用します。
呼び出し側で認可を再実装しません。

![図6：契約配布と実データ通信](/images/amplify-gen2-cross-app-type-safe-data/fig-05-distribution-vs-data-path.png)

### 認証

ブラウザからの利用では、Cognito User Poolを使用し、`authMode: "userPool"`を指定します。
（本件では、同一CognitoユーザープールであったためCognito認証が使えた）

Lambdaなどのサーバー間処理では、Lambda実行ロールの一時資格情報による[IAM認証](https://docs.aws.amazon.com/appsync/latest/devguide/security-authz.html)を使用し、`authMode: "iam"`を指定します。

IAM認証で公開するモデルまたは操作には、提供側のSchemaで[`allow.authenticated("identityPool")`](https://docs.amplify.aws/react/build-a-backend/data/customize-authz/signed-in-user-data-access/)を設定します。
Lambda実行ロールには、対象の提供側AppSyncで必要なGraphQLフィールドに対する`appsync:GraphQL`権限を付与します。
つまりIAM認証で実行できる操作は、提供側のSchemaに定義した認可規則と、Lambda実行ロールに付与した`appsync:GraphQL`権限の両方によって制御されます。

契約パッケージでは認証方式を固定せず、認証方式は呼び出し元が指定します。
独自の認証型は定義せず、Amplify標準の`CommonPublicClientOptions`を受け取ります。
接続先の`endpoint`は環境ごとの`PROVIDER_GRAPHQL_ENDPOINT`からBackendへ渡し、ブラウザにはAmplifyのcustom outputs、Lambdaには関数の環境変数として提供します。

呼び出し側が実行できる操作は、提供側のSchemaに定義したモデル単位および操作単位の認可規則で決まります。
参照だけを許可するモデルには、読み取りのみを許可する認可規則を設定します。
呼び出し側アプリごとに実行できる操作を分ける場合は、提供側AppSyncの[Lambda authorizer](https://aws.amazon.com/blogs/mobile/appsync-lambda-auth)を使用し、呼び出し側は`authMode: "lambda"`を指定します。
Lambda authorizerを追加する場合も、契約パッケージの収録内容はそのまま使用します。

![図7：認証と認可](/images/amplify-gen2-cross-app-type-safe-data/fig-06-auth-and-authorization.png)

### クライアントの分離

呼び出し側自身のAppSyncへ接続する内部Data clientと、提供側のAppSyncへ接続するData clientは、別々に生成します。
接続先もSchemaも異なるため、二つのData clientを一つへ統合しません。

とはいえ、両方のData clientでmodels、queries、mutations、subscriptionsという同じ利用方法を使用します。
接続先ごとに異なるAPI操作方法を使用しません。

関連取得の深さは提供側で固定せず、呼び出し側が操作ごとに`selectionSet`で指定します。
呼び出し側が取得できる関連モデルやフィールドは、提供側のSchemaに定義された認可規則と、リクエストに使用する認証方式によって制御します。

## 実装時に発生した問題

### 外部endpoint用clientのruntime実装

Amplifyの`generateClient`へ`Schema`の型引数と外部`endpoint`を指定すると、型上は`client.models`が存在します。
ただ、実行時のモデル実装は構築されないため、`client.models`の呼び出しは実行時に失敗します。
型検査は成功するので、原因の特定に時間を要しました（型検査に成功するという事実そのものが、原因の特定を遅らせました）。

解決には、`createProviderClient`の内部でruntime introspectionを追加する構成を使用します。

この問題は型検査では検出されず、実行時に表面化します。
外部連携を型情報だけで構築できたように見える場合も、実行時の動作確認が必要です。

### CodeArtifact認証の実行時点

契約パッケージのインストールは、`npm ci`の実行時に行われます。
`backend.ts`が評価されるのは、依存関係のインストールが完了した後です。
つまり、CodeArtifactへの認証を`backend.ts`の内部で実行することはできません。
認証が完了していない場合、アプリケーションコードの実行前に`npm ci`が認証エラーで失敗します。

そこで[Amplify Hosting](https://docs.amplify.aws/react/deploy-and-host/fullstack-branching/custom-pipelines/)では、`amplify.yml`のpreBuildフェーズで[CodeArtifactへログイン](https://docs.aws.amazon.com/codeartifact/latest/ug/npm-auth.html)し、その後に`npm ci`を実行します。

```yaml:amplify.yml
backend:
  phases:
    preBuild:
      commands:
        # npm ciより前にCodeArtifactへログインする
        - aws codeartifact login --tool npm --domain your-domain --repository provider-contract-${AWS_BRANCH}
    build:
      commands:
        - npm ci
        - npx ampx pipeline-deploy --branch $AWS_BRANCH --app-id $AWS_APP_ID
```

Amplify Hostingのサービスロールには、同一アカウント内の対象CodeArtifactリポジトリに対する読み取り権限を、IAMポリシーとして事前に付与します。
整合性検査に使用する`appsync:ListTagsForResource`権限も同じIAMポリシーへ含めます。
提供側AppSyncに対するビルド用ロールの権限はタグ取得に限定し、`appsync:GraphQL`などの実行権限は付与しません。

呼び出し側のSandboxをローカルで起動する開発者にも、権限設定が必要です。
開発者が使用するAWS CLIプロファイルに、`dev`用CodeArtifactリポジトリの読み取り権限と、提供側の`dev`環境にあるAppSyncへの`appsync:ListTagsForResource`権限を付与します。
開発者は`npm ci`の前に`aws codeartifact login`を実行します。

ただ、[CodeArtifactへのログインには有効期限があります](https://aws.amazon.com/blogs/devops/publishing-private-npm-packages-aws-codeartifact)。
有効期限が切れると`npm ci`が認証エラーで失敗するので、その場合はCodeArtifactへ再度ログインします。
再ログイン手順をチームで共有しておくと、認証エラーに関する問い合わせを削減できます。

## 契約と接続先の整合性検査

### 検出する二つの問題

契約パッケージの配布とAppSyncへの接続を構築した後も、二種類の不整合が発生する可能性があります。

| 不整合 | 原因 | 検査 |
| --- | --- | --- |
| 契約パッケージと接続先Schemaが対応していない | 呼び出し側は公開済みの任意のバージョンをインストールでき、契約パッケージだけでは接続先環境の契約を判定できません | AppSyncタグの照合 |
| Schemaを変更したが契約パッケージのバージョンを更新していない | バージョンを変更しなければ、契約パッケージと接続先AppSyncのタグが同じ値を維持します | tarball integrityの比較 |

:::message

単純なバージョン比較では、Schemaの変更とバージョン更新漏れを同時に検出できません。

それならば、二つの問題を別々の検査で検出します。
不整合を検出した場合は、Sandboxの起動、Amplifyのデプロイ、または契約パッケージの公開を停止します。

:::

### AppSyncタグの照合

#### 提供側の処理

提供側の`backend.ts`は、契約パッケージの`package.json`を直接参照し、契約パッケージの名称とバージョンを`provider-contract`タグとしてAppSync APIへ付与します。

AppSyncにはGraphQL Schemaのバージョンという概念がありません。
そこで`provider-contract`タグは、そのAPIへデプロイされている契約パッケージの識別子として扱います。

#### 呼び出し側の処理

呼び出し側の`backend.ts`は、ルート`package.json`で完全一致指定した契約パッケージの名称とバージョンを期待値とします。
その上で接続先AppSyncのタグを取得し、期待値と文字列で比較します。

```typescript:amplify/backend.ts
// 呼び出し側のbackend.tsにある主要処理
const expected = `${contractName}@${contractVersion}`;

const { tags } = await appsync.listTagsForResource({
  resourceArn: providerApiArn,
});

if (tags?.["provider-contract"] !== expected) {
  throw new Error(
    `契約の不一致: 期待${expected} / 接続先${tags?.["provider-contract"]}`,
  );
}
```

値が一致しない場合は、Sandboxの起動とAmplifyのデプロイを停止します。

#### 期待値の取得元

期待値は、照合専用の設定から取得しません。
契約バージョンを独自の環境変数で配布すると、AppSyncのタグ、`package.json`のバージョン、環境変数の三つを管理する必要があります。
三つの値が一致しない可能性が生じるため、この案は採用しません。

照合には、ルート`package.json`の完全一致指定と、接続先AppSyncの`provider-contract`タグだけを使用します。
整合性検査を実行する呼び出し側のBackendまたはビルド用ロールが提供側AppSyncへ必要とする権限は、[`appsync:ListTagsForResource`](https://docs.aws.amazon.com/appsync/latest/APIReference/API_ListTagsForResource.html)だけに限定します。

### tarball integrityの比較

#### 検査方法

提供側のCIは、Backendのデプロイ後に契約パッケージの[tarball](https://docs.npmjs.com/cli/v10/commands/npm-pack)を作成し、その[integrity](https://docs.npmjs.com/cli/v10/configuring-npm/package-lock-json)と、同一バージョンで公開済みのtarballのintegrityを比較します。

integrityが異なる場合は、同じバージョンのままパッケージの内容が変更されたと判定し、バージョン更新を要求してCIを失敗させます。
公開済みのバージョンは変更不可として扱い、同一バージョンを再公開しません。

#### CIの実行順序

runtime introspectionは、デプロイ済みAppSyncのoutputsから生成し、tarball integrity検査の材料にも含まれます。
このため、tarball integrityの比較をAppSyncのデプロイ前に実行することはできません。

CIの実行順序は次のとおりとします。

1. Backendをデプロイします。
2. デプロイ済みAppSyncのoutputsを生成します。
3. 契約パッケージのtarballを作成します。
4. 公開済みtarballとのintegrityを比較します。
5. integrityが一致した場合だけ契約パッケージを公開します。

**この処理順序は変更できない制約です。**

![図8：契約パッケージ公開CI](/images/amplify-gen2-cross-app-type-safe-data/fig-07-contract-publish-ci.png)

#### 比較対象

integrityの比較対象は、契約パッケージへ収録される配布物全体です。

- 提供側の全Schema型
- 全model introspection
- 型宣言
- `createProviderClient`
- README
- 契約パッケージへ実際に収録されるその他のファイル

![図9：二段構えの整合性検査](/images/amplify-gen2-cross-app-type-safe-data/fig-08-consistency-checks.png)

## ブランチごとに分離した環境

Amplify Hostingでは、`dev`、`staging`、`main`の3ブランチで環境を運用します。
Backendは、ブランチ名からCodeArtifactのドメインとリポジトリを動的に作成します。

各ブランチでは、そのブランチへデプロイしたAppSyncのoutputsから契約パッケージを生成し、ブランチに対応するCodeArtifactリポジトリへ公開します。
呼び出し側の各Hosting環境は、対応する提供側の環境とCodeArtifactリポジトリだけに接続します。
呼び出し側のローカルSandboxと`dev`環境は、提供側の`dev`環境へ接続し、`dev`用CodeArtifactリポジトリから契約パッケージを取得します。

**環境をまたぐ接続とパッケージ取得は行いません。**
確認済みの変更は、Gitで`dev`、`staging`、`main`の順にマージします。

![図10：ブランチごとに分離した環境](/images/amplify-gen2-cross-app-type-safe-data/fig-09-per-branch-environments.png)

## 契約バージョンの運用

### 契約バージョンの完全一致指定

**呼び出し側は、`^1.2.3`のような範囲指定を使用せず、`1.2.3`のように契約パッケージのバージョンを完全一致で指定します。**

範囲指定では、`npm ci`の実行時点によって取り込まれる契約が変わる可能性があります。
取り込まれる契約が変わると、同一コミットから同一ビルドを再現できません。
完全一致で指定すると、ビルドに使用した契約が`package.json`とロックファイルに記録されます。

契約の更新は、呼び出し側の明示的なコミットによって行います。
つまり、契約の変更をコードレビューの対象にできます。

### 呼び出し側への契約更新

契約パッケージのバージョンを固定する場合、提供側が新しい契約を公開しただけでは呼び出し側の型は変更されません。
呼び出し側は、通常の開発フローで契約を更新します。

1. `package.json`のバージョン指定を更新し、契約パッケージをインストールします。
2. 発生したコンパイルエラーを解消します。
3. ローカルSandboxから提供側の`dev`環境へ接続し、動作を確認します。
4. `dev`、`staging`、`main`の順にマージします。

提供側のSchemaを先に更新すると、呼び出し側が追従するまで新しいSchemaと古い呼び出しコードが併存します。
デプロイ済みの呼び出し側は、最後にビルドした契約で動作を継続します。
併存する期間は、リリース順序と通知によって管理します。

### 破壊的変更の手順

削除、名称変更、型変更は、次の順序で実施します。

1. 新しい項目を追加します。
2. 新しい契約を公開します。
3. 各呼び出し側を新しい項目へ移行します。
4. 旧項目が使用されていないことを確認します。
5. 旧項目を削除します。

提供側のSchema変更を、連携用APIの変換処理で隠す方式ではありません。
新しい型は、契約として呼び出し側へ伝播します。
移行期間中は、旧フィールドと旧操作をSchemaに残します。

AppSyncは、一つのAPIに対して複数バージョンのSchemaを同時に提供できません。
そのため、フィールドの追加、呼び出し側の移行、旧フィールドの削除という順序で段階的に変更します。

![図11：破壊的変更の段階移行](/images/amplify-gen2-cross-app-type-safe-data/fig-10-breaking-change-migration.png)

:::details 採用しなかった方式

検討後に採用しなかった代表的な方式と理由は、次のとおりです。

| 方式 | 見送った理由 |
| --- | --- |
| Schema型だけをnpm契約パッケージとして配布 | 型上は`client.models`が存在しますが、外部`endpoint`へ接続するクライアントの実行時実装が構築されず、実行できません。 |
| 提供側リポジトリをcloneして`resource.ts`を直接参照 | 公式例にある構成ですが、すべての呼び出し側のローカル環境とHostingのビルドへ、clone、Git認証、固定配置が必要になります。参照するブランチの先頭とデプロイ済みAPIが一致しない可能性があります。 |
| 複数のAmplifyアプリを一つのモノレポへ統合し、提供側の`resource.ts`を直接参照 | 対象のアプリは既に本番運用を開始しており、担当する開発者とリリース時期が異なります。統合には稼働中のアプリの構成変更が必要になります。リポジトリ上の`resource.ts`と、デプロイ済みAPIが一致しない可能性も残ります。 |
| 提供側のDynamoDBを呼び出し側AppSyncのデータソースにする | キー以外の属性型が保持されません。項目追加時にSchemaへの目視転記とResolverの再実装が必要になります。提供側の認可を経由しません。 |
| [AppSync Merged API](https://docs.aws.amazon.com/appsync/latest/devguide/merged-api.html) | 提供側AppSyncから統合APIまでの処理は自動化されますが、呼び出し側の型取得は別途必要になります。アプリ間連携に対して構成が過剰になります。 |

表に記載した各方式には、追加の判断材料があります。

まず、Schema型だけを配布する方式も検討しました。
ただ、Schema型だけでは実行時のモデル実装が構築されないため、採用しませんでした。
現在の契約パッケージには、型だけでなくruntime introspectionとData clientの生成関数を含めます。

提供側リポジトリをcloneして参照する方式は、公式の例にも含まれます。
とはいえ、呼び出し側が複数存在する環境では、すべての呼び出し側のローカル開発環境とHostingのビルドでcloneを実行する必要があります。
すべての呼び出し側でGit認証とリポジトリの固定配置が必要になりますし、参照するブランチの先頭と、実際にデプロイされているAPIが一致しない可能性もあります。
現在の構成では、提供側の`resource.ts`に定義された`Schema`を契約の正本とし、runtime introspectionはリポジトリ上のブランチではなく、デプロイ済みAppSyncのoutputsから生成します。
この構成により、型の正本を明確にしたまま、実際に稼働しているAPIのruntime情報を契約パッケージへ収録します。

DynamoDBの直接参照方式は、中間APIを使用しない構成です。
中間APIを使用しないことだけを基準にすると、これが最短の構成になります。
ただ、型を自動取得できず、提供側の認可も使用できません。
データ経路の短縮と引き換えに、型の自動取得と提供側の認可を失います。

[AppSync Merged API](https://aws.amazon.com/blogs/mobile/introducing-merged-apis-on-aws-appsync/)は、AWSが提供するSchema統合機能で、最終段階まで比較対象としました。
自動化する範囲は提供側AppSyncから統合APIまでで、呼び出し側が型を取得する仕組みは別に用意する必要があります。
つまり、呼び出し側へ型を配布する問題は解決できません。
統合APIという追加リソースを運用しても、型配布の仕組みは別途必要になります。
結局、統合APIを追加せず、型配布の仕組みだけを構築する方針を採用しました。

採用しなかった方式は、型の自動伝播または実データ経路の単純性のいずれかを満たしませんでした。
契約パッケージと提供側AppSyncへの直接接続を組み合わせると、型の自動伝播と実データ経路の単純性を同時に満たせます。

:::

:::details 異なるUser Poolへの対応

現在は、提供側と同じCognito User Poolを使用する呼び出し側で運用しています。
異なるUser Poolを使用する呼び出し側へ対応する場合は、提供側AppSyncへ[追加の認証プロバイダー](https://docs.aws.amazon.com/appsync/latest/devguide/security-authz.html)を登録します。

:::

## まとめ

この構成では、提供側の`Schema`を型の正本に据え、デプロイ済みAppSyncのmodel introspectionとData client生成関数を同じ契約パッケージで配布します。

呼び出し側は外部AppSyncでも`client.models`を使用でき、GraphQLと型定義の手書きを増やさずに済みます。

一方、型が通ることと実行時に動くことは別です。

外部endpoint用clientへruntime introspectionを追加し、AppSyncタグとtarball integrityを照合することで、契約パッケージ、稼働中Schema、配布物のずれをCIで止めます。

低レベルruntime APIへの依存は残りますが、依存箇所は契約パッケージの内部へ閉じ込めました。

既存アプリを統合せず、複数リポジトリのまま開発体験を揃えるために選んだ構成です。
