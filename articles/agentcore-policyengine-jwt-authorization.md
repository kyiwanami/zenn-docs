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
published_at: 2026-03-09
---

## はじめに

業務アプリにエージェントを組み込んでいくと、運用・セキュリティの観点で「モデルに見せるツールをどう絞るか」「許可されていない操作をどう実行不能にするか」が論点になります。

今回は Amplify でホストした業務アプリを前提に、Cognito の JWT claims と Amazon Bedrock AgentCore PolicyEngine を使って、エージェントの認可を制御した構成を紹介します。

ハッカソンで作った全体像は[こちらの記事](こちらの記事)にまとめています。

> 本記事の内容は記事執筆時点の Amazon Bedrock AgentCore の仕様を前提にしています。プレビュー中の機能やコンソール仕様は今後変更される可能性があるため、実運用で採用する際は最新ドキュメントを確認してください。

## 今回の全体構成

今回の構成では、次の流れで認可判定を行います。

1. Cognito group でユーザーを分類する
2. JWT claims に role を入れる
3. Amplify frontend から AgentCore Runtime を呼ぶ
4. AgentCore Runtime から AgentCore Gateway にアクセスする
5. AgentCore Gateway が PolicyEngine に問い合わせる
6. `tools/list` で見せてよいツールだけを返す
7. `tools/call` で実際に呼んでよい操作だけを通す

![今回の全体構成](https://assets.st-note.com/img/1772552236-81QyxFztWCsB2PvYpqlEVim3.png?width=1200)

この構成で重要なのは、Gateway 側で「一覧に見せるか」と「実際に呼ばせるか」の両方を評価して絞ることです。モデルにツール名が見えていなくても困りますし、逆にツール名を知っていても実行できてはいけません。そのため `tools/list` と `tools/call` の両方で PolicyEngine を通しています。

参考: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-core-concepts.html

## PolicyEngineとは

Amazon Bedrock AgentCore PolicyEngine は、Cedar 言語で記述したポリシーに基づいて、リクエストを許可するかどうかを判定する仕組みです。

Cedar では、どの principal が、どの action を、どの resource に対して行えるかをポリシーとして宣言的に書けます。たとえば `refund` という tool に対するポリシーは、次のように書けます。

```cedar
permit (
  principal,
  action == Action::"tool:call",
  resource == Tool::"refund"
)
when {
  principal.role == "manager"
};
```

また PolicyEngine には `LOG_ONLY` と `ENFORCE` というモードがあります。`LOG_ONLY` は結果をログに残すだけで実際の拒否は行わず、`ENFORCE` は判定結果を実際の制御に使います。まずは `LOG_ONLY` で意図した判定になっているかを確認し、その後 `ENFORCE` に切り替える流れが安全です。

関連URL:

- https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy-engine.html
- https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-authz.html
- https://www.cedarpolicy.com/

## 認可をPolicyEngineでどう判定したか

今回は Cognito の group を `viewer` / `editor` / `manager` に分け、ログインしたユーザーの所属 group に応じて JWT claims に `role` を入れるようにしました。

![Cognito group と JWT claims](https://assets.st-note.com/img/1772552445-67ZDselxU8BCprTGyigc90KF.png)

AgentCore Gateway にはフロントエンドから JWT を渡せるため、その claims をそのまま PolicyEngine の判定材料に使えます。つまり「アプリ側で認可を判定して UI を変える」だけではなく、「Gateway 側でも同じ属性を使って本当に実行可能かを判定する」という二重の制御ができます。

JWT claims の付与には Cognito の pre token generation trigger を使いました。サインイン時に group 情報を見て `role` claim を埋めることで、後段の AgentCore 側は claims を見るだけで判定できます。

関連URL:

- https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html
- https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-id-token.html

## やりたいことのマトリクス

今回やりたかったのは、ロールによって使えるツールを明確に分けることです。

![やりたいことのマトリクス](https://assets.st-note.com/img/1772552602-cNI4v8ndU9ZDieSKWjhzL7k5.png)

たとえば `delete-asset` は manager のみが実行できるようにし、それ以外のロールでは許可しないポリシーを Cedar で書きます。

```cedar
permit (
  principal,
  action == Action::"tool:call",
  resource == Tool::"delete-asset"
)
when {
  principal.role == "manager"
};
```

## 実行可能ツール取得の結果が権限属性で変わること

実際に `tools/list` を叩くと、role に応じて取得できるツール一覧が変わります。

manager では create / update / delete を含むツールがすべて見えます。

![manager のツール一覧](https://assets.st-note.com/img/1772552917-lTZwcWvOrG4a6dbzFpDP0MYS.png?width=1200)

viewer では create / update / delete 系が見えず、閲覧系だけが返ってきます。

![viewer のツール一覧](https://assets.st-note.com/img/1772552986-1YSmdtOMGhB8JpAnCuQcVb9R.png?width=1200)

Gateway 側のログでも、claims に応じて返却対象が絞られていることが確認できます。

```text
[Gateway] authorization check for tools/list
principal.role=manager
allowedTools=["list-asset","get-asset","create-asset","update-asset","delete-asset"]

[Gateway] authorization check for tools/list
principal.role=viewer
allowedTools=["list-asset","get-asset"]
```

## ツール名を明示しても実行できないこと

次に、ツール一覧に見えていなくても、ツール名を直接指定すれば呼べてしまうのではないかを確認しました。

viewer で `delete-asset` を明示して実行すると、JSON-RPC のレスポンスは拒否になります。

![delete-asset を直接指定した結果](https://assets.st-note.com/img/1772553033-bKm2ECBUd4ufR81PytMogTI9.png?width=1200)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32603,
    "message": "tool call is not authorized: delete-asset"
  }
}
```

Gateway ログでも `tools/call` 側で拒否されていることが分かります。

```text
[Gateway] authorization check for tools/call
principal.role=viewer
tool=delete-asset
decision=DENY
mode=ENFORCE
```

一方、許可されたユーザーであれば同じ `delete-asset` を実行できます。

![manager で delete-asset を実行した結果](https://assets.st-note.com/img/1772553068-BIoelvSHpjaVm0T5k4ARs1NQ.png)

## まとめ

JWT claims を PolicyEngine で制御する構成は、実運用を見据えた細かい認可に相性が良いと感じました。UI 上で見せるツールを絞るだけでなく、実際の実行可否も Gateway 側で強制できるため、業務アプリにエージェントを組み込む際の安全性を上げやすいです。

~~記事執筆時点では AgentCore policy は preview 扱いですが、~~ 今後CDKの拡充などに期待しています。

追記: 2026/3/3 に GA されました。

https://aws.amazon.com/jp/about-aws/whats-new/2026/03/policy-amazon-bedrock-agentcore-generally-available/
