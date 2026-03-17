---
title: "業務アプリのエージェント認可を、Amazon Bedrock AgentCore PolicyEngineとJWTで設計した"
emoji: "🛡️"
type: "tech"
topics:
  - aws
  - amazonbedrock
  - agentcore
  - policyengine
  - jwt
  - cognito
published: false
publication_name: "trust"
published_at: 2026-03-09
---

こんにちは。エンジニアの岩波です。

最近はエージェントを業務アプリに組み込む動きが盛んですね。私もNotionなどエージェント付き業務効率化アプリに日々助けられています。

一方で、現実のエージェント導入では運用やセキュリティ面が重視されてきています。モデルに見せてよいツールをどう絞るか、許可されていない操作をどう実行不能にするか、既存の認証・認可基盤とどう整合させるかは、本番運用では避けて通れない論点です。

AWS でも、こうした業務アプリ向けのエージェント実装を支える基盤として Amazon Bedrock AgentCore が提供されています。
その中で、AgentCore Gateway におけるツール認可を扱う仕組みが **PolicyEngine** です。

この記事では、AWS Amplifyでホストする業務アプリを前提に、Amazon Cognitoの**JWT claims**と、**PolicyEngine**で**エージェント認可制御**をした構成を紹介します。

尚、本記事内で用いたアプリは、先日社内で開催されたハッカソンで作成しました。
[**こちらの記事**](https://note.com/trust_dx/n/n5ca5c7dc5a94)にハッカソンの様子が掲載されていますので、ぜひご覧ください！

> **⚠️ 注意事項**
>
> 本記事で紹介するコード例は**概念的な実装例**であり、そのまま本番環境で使用できる完全なコードではありません。また、手元で動作したコードをブログ用に簡略化・修正しているため、実際のプロジェクトでは適切なエラーハンドリング、ログ出力、セキュリティ対策などを追加する必要があります。

## 今回の全体構成

今回の構成では、Amazon Cognito groupから付与した権限属性をJWT claimsに反映したうえで、同じJWTをAmplifyフロントエンド、AgentCore Runtime、AgentCore Gatewayの間で貫通させる構成にしています。

![](https://assets.st-note.com/img/1772552236-81QyxFztWCsB2PvYpqlEVim3.png?width=1200)

構成の流れは次のとおりです。

1. Cognito groupから判定用の権限属性を付与する
2. ログイン時のJWTに、判定用のclaimsを持たせる
3. AmplifyフロントエンドがBearer JWT付きでAgentCore Runtimeを呼ぶ
4. Runtimeが同じBearer Tokenを使ってAgentCore Gatewayに接続する
5. GatewayにアタッチされたPolicyEngineがCedar policyで評価する
6. Gateway が tools/list と tools/call を受ける
7. 許可されたツール（Target）だけが取得/実行される

この構成で重要なのは、Gateway 側で 一覧に見せるか と 実際に呼ばせるか の両方を評価して絞っている点です。

> 参考：
> [https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-core-concepts.html](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-core-concepts.html)

## PolicyEngineとは

**PolicyEngine** はエージェントのツール呼び出しを評価して認可するためのポリシーの集合として説明されています。
**Cedar**というAWSによって認可のユースケースをサポートすることを目的として構築された言語を使用します。

AWS公式ドキュメントのサンプルでは、refund toolに対して「username=refund-agent で、かつ amount < 500 のときだけ許可する」ポリシーが紹介されています。

```
permit(
  principal is AgentCore::OAuthUser,
  action == AgentCore::Action::"RefundTool__process_refund",
  resource == AgentCore::Gateway::"arn:aws:bedrock-agentcore:us-west-2:123456789012:gateway/refund-gateway"
)
when {
  principal.hasTag("username") &&
  principal.getTag("username") == "refund-agent" &&
  context.input.amount < 500
};
```

このpolicyをPolicyEngineに入れると、「refund」ツールはusername=refund-agentかつamount < 500のときだけ情報取得/実行が可能です。

> 参考：
> [https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-understanding-cedar.html](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-understanding-cedar.html)
> [https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-natural-language.html](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-natural-language.html)

PolicyEngine には 2 つの enforcement mode があります。
**LOG_ONLY** は allow / deny を評価してログやトレースに残すだけで、実際のブロックはしません。
**ENFORCE** は評価結果をそのまま適用し、許可された操作だけを通します。
まずは前者で効くことを確かめてから、後者の導入をするのが良いと思います。（AWS WAFもこんな思想がありましたね）

> 参考：
> [https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-enforcement-modes.html](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-enforcement-modes.html)

## 認可をPolicyEngineでどう判定したか

![](https://assets.st-note.com/img/1772552445-67ZDselxU8BCprTGyigc90KF.png)

今回の構成では、「viewer」「editor」「manager」といったCognito groupをユーザーに割り当てることで権限を管理しています。

そして、この情報をJWT claimsに載せてGatewayまで渡しています。これはpre token generation triggerで、JWTにrole=viewer / role=editor / role=managerのような判定用claimsを入れる形で実現できます。

> 参考：
> [https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-user-groups.html](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-user-groups.html)
> [https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html)

やりたいことを先にマトリクスで書くと、イメージは次のようになります。

![](https://assets.st-note.com/img/1772552602-cNI4v8ndU9ZDieSKWjhzL7k5.png)

ここでは、Asset という資産データモデルを扱う前提で、閲覧、作成、更新、削除の権限をロールごとに段階的に分けています。

例えば、実装上managerにだけ許可している delete-asset ツールのpolicyは、概念的には次のようになります。
JWT claimsが**principal**の**tags**として扱われ、Cedar policyからprincipal.hasTag(...)やprincipal.getTag(...)で参照されます。

```
permit(
  principal is AgentCore::OAuthUser,
  action == AgentCore::Action::"delete-asset___delete-asset",
  resource == AgentCore::Gateway::"arn:aws:bedrock-agentcore:ap-northeast-1:123456789012:gateway/workops-example-gateway"
)
when {
  principal.hasTag("role") &&
  principal.getTag("role") == "manager"
};
```

## 実行可能ツール取得の結果が権限属性で変わること

まず、実行可能ツール取得（tools/list）の結果そのものがユーザー権限属性で変わることを確認します。

manager ユーザーではすべてのツールが取得される一方、viewer ユーザーでは create-asset、update-asset、delete-asset が tools/list の段階で除外され、エージェントに見えない、ということです。

まず、manager ユーザー側で画面からエージェントを呼び出すと、全てのツールが取得できました。

![](https://assets.st-note.com/img/1772552917-lTZwcWvOrG4a6dbzFpDP0MYS.png?width=1200)

Gateway ログを最小限だけ抜粋すると、例えば次のようになります。

```
[INFO] Received request for tools/list method
resource: arn:aws:bedrock-agentcore:ap-northeast-1:<ACCOUNT_ID>:gateway/<GATEWAY_NAME>
request_id: <REQUEST_ID>

[INFO] Successfully processed request
resource: arn:aws:bedrock-agentcore:ap-northeast-1:<ACCOUNT_ID>:gateway/<GATEWAY_NAME>
result.tools:
- list-assets___list-assets
- create-asset___create-asset
- update-asset___update-asset
- delete-asset___delete-asset
```

次に、viewer ユーザー側で実行します。
create-asset、update-asset、delete-assetは出ておらず、期待通りです。

![](https://assets.st-note.com/img/1772552986-1YSmdtOMGhB8JpAnCuQcVb9R.png?width=1200)

ログは次のようになっています。

```
[INFO] Received request for tools/list method
resource: arn:aws:bedrock-agentcore:ap-northeast-1:<ACCOUNT_ID>:gateway/<GATEWAY_NAME>
request_id: <REQUEST_ID>

[INFO] Successfully processed request
resource: arn:aws:bedrock-agentcore:ap-northeast-1:<ACCOUNT_ID>:gateway/<GATEWAY_NAME>
result.tools (asset-related excerpt):
- list-assets___list-assets
```

## ツール名を明示しても実行できないこと

ただし、一覧から消えるだけでは十分ではありません。
ユーザーやモデルがツール名を知っていて、プロンプトで直接指定してきた場合でも、実行できない必要があります。

そこで、制限されたユーザーで manager 向けツール名（delete-asset）を明示した入力を行い、実際に実行できないことも確認しました。

![](https://assets.st-note.com/img/1772553033-bKm2ECBUd4ufR81PytMogTI9.png?width=1200)

エージェントが参照できないだけでなく、フロントエンドからGatewayツールを直接呼び出した場合も、Gateway側で拒否されることを確認しました。

```
{
  "jsonrpc": "2.0",
  "id": "call-delete-asset",
  "error": {
      "code": -32002,
      "message": "Tool Execution Denied: Tool call not allowed due to policy enforcement [No policy applies to the request (denied by default).]"
  }
}
```

```
[ERROR] Tool Execution Denied: Tool call not allowed due to policy enforcement [No policy applies to the request (denied by default).]
resource: arn:aws:bedrock-agentcore:ap-northeast-1:<ACCOUNT_ID>:gateway/<GATEWAY_NAME>
request_id: <REQUEST_ID>
trace_id: <TRACE_ID>
severity: ERROR
```

もちろん、許可されたユーザーであれば問題なくツール実行できます。

![](https://assets.st-note.com/img/1772553068-BIoelvSHpjaVm0T5k4ARs1NQ.png)

## まとめ

業務アプリにエージェント機能を持たせるとき、Amazon Cognito の JWT claims を PolicyEngine で制御する構成は、実運用を見据えた細かい認可の持たせ方として相性が良いと感じました。

業務アプリの既存 JWT をそのままエージェントのツール認可へ接続できる構成としてはかなり面白く、実運用に向けた方向性は見えたと思います。
記事執筆時点では AgentCore policy は preview 扱いですが、今後CDKの拡充などに期待しています。

> 追記
> **このタイミングでちょうどGAされたようです！！**（2026/3/3）
> https://aws.amazon.com/jp/about-aws/whats-new/2026/03/policy-amazon-bedrock-agentcore-generally-available/
