---
title: "Amplify Gen2とCognito Hosted UIで実現する！アプリ間SSOガイド"
emoji: "🔐"
type: "tech"
topics: ["AWS", "Amplify", "Cognito", "SSO", "React", "TypeScript"]
published: false
publication_name: "trust"
published_at: 2025-07-29
---

### はじめに

こんにちは。株式会社トラストの岩波です。

弊社のプロジェクトで、元々Amplify Gen2で構築した複数の業務アプリケーションがあり、それらを横断的に利用するためのポータルサイトを新規で作りたい、という要件が持ち上がりました。

その際、ユーザー体験を損なわないために、一度のログインで全てのアプリケーションをシームレスに利用できるシングルサインオン（SSO）の導入が不可欠でした。

既存のアプリケーションがすべてAWS Amplify Gen2で構築されていたため、SAMLやOpenID Connect (OIDC)をゼロから設定するよりも、Amplifyの標準機能であるCognitoを最大限に活用する方が、シンプルかつ迅速に実現できると判断しました。

この記事では、AWS Amplify Gen2とAmazon CognitoのHosted UIを用いて、複数のAmplifyアプリケーション間でSSOを実現する方法を、具体的なコードを交えながらステップバイステップで解説します。

## 成果物

ユーザーを作成し、一方のアプリでログインしておきます。

![](https://assets.st-note.com/img/1769767724-p2DdxjPOEheBFYyZ0TWgrMKA.png?width=1200)

![](https://assets.st-note.com/img/1769767736-EUpOnjW3X5DPVQqLKvboAaxI.png?width=1200)

もう一方のアプリにアクセスすると、改めてログインしていないのに同一ユーザーとして認識されています。

![](https://assets.st-note.com/img/1769767748-yQSkVm8DciMx92hpgfAlL1dH.png?width=1200)

今回は、この動きができることを目指します。

## システム構成

**今回構築するシステムの構成は以下の通りです。**

- **親アプリ (Auth Provider):** 認証情報を一元管理する中心的なAmplifyアプリ。Cognitoユーザープールを所有します。
- **子アプリ (Auth Consumer):** 親アプリのCognitoユーザープールを利用して認証を行う、既存または新規のAmplifyアプリ。

![](https://assets.st-note.com/img/1769767772-6zalTUWjN5CseInwXOQH0Fvu.png?width=1200)

### 前提条件

- Node.jsとnpmがインストールされていること
- AWSアカウントと、Amplifyを操作するためのIAMユーザーが設定済みであること
- React（Vite）とTypeScriptの基本的な知識があること
- Amplify Gen2アプリのデプロイが完了していること
    - 今回は公式ドキュメントのクイックスタートを実施しましたhttps://docs.amplify.aws/react/start/quickstart/

## 手順

### 1. 親アプリ（認証サーバー）の設定

まず、認証の親となるAmplifyアプリを準備し、Cognito Hosted UIを利用するための設定を行います。

### package.json

まず、必要なライブラリが指定のバージョン以上であることを確認してください。

```
{
  "dependencies": {
    ...
    "aws-amplify": "^6.12.2",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
    ...
  },
  "devDependencies": {
    ...
    "@aws-amplify/backend": "^1.14.3",
    "@aws-amplify/backend-cli": "^1.4.2",
    "@types/node": "^24.0.1",
    "@types/react": "^18.2.66",
    "@types/react-dom": "^18.2.22"
    ...
  }
}
```

### amplify/auth/resource.ts

認証リソースの定義です。ここでは基本的なメールアドレスでのログイン設定のみで、変更は不要です。

```
import { defineAuth } from '@aws-amplify/backend';

/**
 * Define and configure your auth resource
 * @see https://docs.amplify.aws/gen2/build-a-backend/auth
 */
export const auth = defineAuth({
  loginWith: {
    email: true,
  },
});
```

### amplify/backend.ts

ここが最も重要な設定ファイルです。Cognito Hosted UIを有効化し、SSO対象となるアプリ（子アプリ）のURLをコールバックURLとして登録します。

```
import { defineBackend } from '@aws-amplify/backend';
import { auth } from './auth/resource';
import { data } from './data/resource';

const backend = defineBackend({
  auth,
  data,
});

const { userPool } = backend.auth.resources;

// CognitoのドメインプレフィックスはAWSアカウント・リージョン内でグローバルに一意である必要があるため、
// 本番環境（prod）と開発環境（sandbox）でプレフィックスを分けて、名前の衝突を回避しています。
const isSandbox = backend.stack.stackName.includes('-sandbox');
const domainPrefix = isSandbox ? 'trust-parent-sandbox' : 'trust-parent';

// カスタムドメイン（Cognito Hosted UIのURL）を設定
userPool.addDomain('custom-domain', {
  cognitoDomain: {
    domainPrefix
  }
});

const { cfnUserPoolClient } = backend.auth.resources.cfnResources;

// OAuthのスコープを設定
cfnUserPoolClient.allowedOAuthScopes = [
  'openid',
  'aws.cognito.signin.user.admin'
];

// この設定により、各アプリを疎結合に保てます。
// 関連アプリが増減するたびに親アプリのソースコードを修正・再デプロイすることなく、
// 環境変数を変更するだけで、SSOの対象を柔軟に変更できます。
const TARGET_DOMAIN = process.env.TARGET_DOMAIN?.split(',') || [];

// ログイン後のリダイレクト先URLを設定
cfnUserPoolClient.callbackUrLs = [
  'http://localhost:5173/', // ローカル開発用
  ...TARGET_DOMAIN,
];

// ログアウト後のリダイレクト先URLを設定
cfnUserPoolClient.logoutUrLs = [
  'http://localhost:5173/', // ローカル開発用
  ...TARGET_DOMAIN,
];

// 認証フローとして認可コードフローを指定
cfnUserPoolClient.allowedOAuthFlows = ['code'];
cfnUserPoolClient.allowedOAuthFlowsUserPoolClient = true;

// amplify-outputs.jsonにCognitoドメインをカスタム出力
const domain = `${domainPrefix}.auth.ap-northeast-1.amazoncognito.com`;
backend.addOutput({
  custom: {
    cognitoDomain: domain,
  }
});

export default backend;
```

**補足
**Cognitoドメインが残存したまま、サンドボックス含むバックエンド環境を削除しようとするとエラーになります。削除する場合は、初めにマネジメントコンソールから設定したCognitoドメインを削除してください。

### src/App.tsx (親アプリのフロントエンド)

親アプリのフロントエンドでも、Amplifyライブラリを設定し、認証状態をハンドリングします。

amplify-outputs.json から読み込んだ設定に対して、リダイレクトURLを現在のオリジン（window.location.origin）で動的に上書きすることで、自分のドメインで正しく動作するようにします。

認証ドメインについては、理由は不明ですがamplify-outputs.jsonのauthブロックに出力されないため、backend.tsでカスタム出力したものを設定しています。

```
import { useEffect, useState } from "react";
import type { Schema } from "../amplify/data/resource";
import { generateClient } from "aws-amplify/data";
import { Amplify } from "aws-amplify";
import outputs from "../amplify_outputs.json";
import { AuthUser, getCurrentUser, signInWithRedirect, signOut } from "aws-amplify/auth";

const client = generateClient<Schema>();

const origin = `${window.location.origin}/`;
outputs.auth.oauth = {
  ...outputs.auth.oauth,
  domain: outputs.custom.cognitoDomain,
  redirect_sign_in_uri: [origin],
  redirect_sign_out_uri: [origin],
}
Amplify.configure(outputs);

function App() {
  const [todos, setTodos] = useState<Array<Schema["Todo"]["type"]>>([]);
  const [user, setUser] = useState<AuthUser | undefined>(undefined);

  useEffect(() => {
    client.models.Todo.observeQuery().subscribe({
      next: (data) => setTodos([...data.items]),
    });
  }, []);

  useEffect(() => {
    const init = async () => {
      try {
        const user = await getCurrentUser();
        setUser(user);
      } catch (error) {
        await signInWithRedirect({ options: { lang: "ja" } });
      }
    };
    init();
  }, []);

  function createTodo() {
    client.models.Todo.create({ content: window.prompt("Todo content") });
  }

  return (
    <main>
      <div>
        <p>親アプリです</p>
        <p>User: {user?.username || 'No user'}</p>
        <button onClick={() => signOut()}>Sign out</button>
      </div>
      <h1>My todos</h1>
      <button onClick={createTodo}>+ new</button>
      <ul>
        {todos.map((todo) => (
          <li key={todo.id}>{todo.content}</li>
        ))}
      </ul>
      <div>
         App successfully hosted. Try creating a new todo.
        <br />
        <a href="https://docs.amplify.aws/react/start/quickstart/#make-frontend-updates">
          Review next step of this tutorial.
        </a>
      </div>
    </main>
  );
}

export default App;
```

### 2. 子アプリ（認証クライアント）の設定

次に、親アプリの認証情報を利用する子アプリを設定します。

### package.json

親アプリと同様に、ライブラリのバージョンを確認します。

```
{
  "dependencies": {
    "aws-amplify": "^6.12.2",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@aws-amplify/backend": "^1.14.3",
    "@aws-amplify/backend-cli": "^1.4.2",
    "@types/node": "^24.0.1"
  }
}
```

### amplify/auth/resource.ts

子アプリでは新しい認証リソースを作成するのではなく、referenceAuth を使って親アプリの既存のCognitoユーザープールを参照します。

必要なIDやARNは、のちほどAmplifyのホスティング環境変数経由で設定します。

```
import { referenceAuth } from "@aws-amplify/backend";

const USER_POOL_ID = process.env.REACT_APP_USER_POOL_ID;
const IDENTITY_POOL_ID = process.env.REACT_APP_IDENTITY_POOL_ID;
const AUTH_ROLE_ARN = process.env.REACT_APP_AUTH_ROLE_ARN;
const UNAUTH_ROLE_ARN = process.env.REACT_APP_UNAUTH_ROLE_ARN;
const USER_POOL_CLIENT_ID = process.env.REACT_APP_USER_POOL_CLIENT_ID;

if (!USER_POOL_ID || !IDENTITY_POOL_ID || !AUTH_ROLE_ARN || !UNAUTH_ROLE_ARN || !USER_POOL_CLIENT_ID) {
  throw new Error("認証に必要な環境変数が設定されていません。");
}

export const auth = referenceAuth({
  userPoolId: USER_POOL_ID,
  identityPoolId: IDENTITY_POOL_ID,
  authRoleArn: AUTH_ROLE_ARN,
  unauthRoleArn: UNAUTH_ROLE_ARN,
  userPoolClientId: USER_POOL_CLIENT_ID,
});
```

### src/App.tsx (子アプリのフロントエンド)

フロントエンドでは、Amplifyライブラリの設定を少しカスタマイズします。

amplify-outputs.json から読み込んだ設定に対して、リダイレクトURLを現在のオリジン（window.location.origin）で動的に上書きすることで、自分のドメインで正しく動作するようにします。

```
import { useEffect, useState } from "react";
import type { Schema } from "../amplify/data/resource";
import { generateClient } from "aws-amplify/data";
import { Amplify } from "aws-amplify";
import outputs from "../amplify_outputs.json";
import { AuthUser, getCurrentUser, signInWithRedirect, signOut } from "aws-amplify/auth";

const client = generateClient<Schema>();

const origin = `${window.location.origin}/`;
outputs.auth.oauth = {
  ...outputs.auth.oauth,
  redirect_sign_in_uri: [origin],
  redirect_sign_out_uri: [origin],
}
Amplify.configure(outputs);

function App() {
  const [todos, setTodos] = useState<Array<Schema["Todo"]["type"]>>([]);
  const [user, setUser] = useState<AuthUser | undefined>(undefined);

  useEffect(() => {
    client.models.Todo.observeQuery().subscribe({
      next: (data) => setTodos([...data.items]),
    });
  }, []);

  useEffect(() => {
    const init = async () => {
      try {
        const user = await getCurrentUser();
        setUser(user);
      } catch (error) {
        await signInWithRedirect({ options: { lang: "ja" } });
      }
    };
    init();
  }, []);

  function createTodo() {
    client.models.Todo.create({ content: window.prompt("Todo content") });
  }

  return (
    <main>
      <div>
        <p>子アプリです</p>
        <p>User: {user?.username || 'No user'}</p>
        <button onClick={() => signOut()}>Sign out</button>
      </div>
      <h1>My todos</h1>
      <button onClick={createTodo}>+ new</button>
      <ul>
        {todos.map((todo) => (
          <li key={todo.id}>{todo.content}</li>
        ))}
      </ul>
      <div>
         App successfully hosted. Try creating a new todo.
        <br />
        <a href="https://docs.amplify.aws/react/start/quickstart/#make-frontend-updates">
          Review next step of this tutorial.
        </a>
      </div>
    </main>
  );
}

export default App;
```

### 3. Amplifyホスティングとドメイン設定

デプロイ済みの各アプリの認証を連携させるための、環境変数の設定手順は以下の通りです。

### 必要な情報を集める

まず、各アプリの設定に必要な情報を集約します。

**親アプリのCognito情報**

amplify-outputs.json から取得: 親アプリのプロジェクトに生成された amplify-outputs.json ファイルから、以下の情報を控えます。

```
{
  "auth": {
    "user_pool_id": "ap-northeast-1_xxxxxxxxx",
     ...
    "user_pool_client_id": "xxxxxxxxxxxxxxxxxxxxxxxxx",
    "identity_pool_id": "ap-northeast-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
     ...
  }
}
```

**AWSマネジメントコンソールから取得**

amplify-outputs.jsonにはIAMロールの情報は出力されないため、AWSコンソールから直接確認します。

- 親アプリのAmplifyコンソールから、認証カテゴリ（Cognito）に移動し、関連付けられた**IDプール（Identity Pool）** を開きます。
- IDプールの「ユーザーアクセス」設定などで、**認証されたロール（Authenticated role）**と**認証されていないロール（Unauthenticated role）**のARNをそれぞれ控えます。これがauthRoleArnとunauthRoleArnになります。

![](https://assets.st-note.com/img/1769767991-J8Wj5oUntPwC7ASlZrTFR6a3.png?width=1200)

**全アプリのURL:**

デプロイ済みの親アプリとすべての子アプリのURLを控えます。

- デプロイ済みの**親アプリとすべての子アプリ**のURLを控えます。

### 子アプリの環境変数を設定する

次に、各子アプリが親の認証情報を使えるように設定します。

Amplifyコンソールで各子アプリの「ホスティング」→「環境変数」を開きます。「変数を管理」をクリックし、手順1で控えた親アプリのCognito情報とIAMロール情報を、amplify/auth/resource.tsで定義した変数名で設定します。環境変数を保存し、各子アプリを再デプロイして設定を反映させます。

- Amplifyコンソールで**各子アプリ**の「ホスティング」→「環境変数」を開きます。
- 「変数を管理」をクリックし、**手順1**で控えた親アプリのCognito情報とIAMロール情報を、amplify/auth/resource.tsで定義した変数名で設定します。
- 環境変数を保存し、各子アプリを再デプロイして設定を反映させます。

![](https://assets.st-note.com/img/1769768023-40r2eOIPfAnS65L3iX1BZ8az.png?width=1200)

**親アプリの環境変数を設定する**

最後に、親アプリのCognitoに、SSOを許可するアプリのURLを登録します。

- Amplifyコンソールで**親アプリ**の「ホスティング」→「環境変数」を開きます。
- 「変数を管理」をクリックし、以下の環境変数を設定します。
    - ◦ **変数:** TARGET_DOMAIN
    - ◦ **値:** **手順1**で控えた**全アプリ（親と子）**のURLをカンマ区切りで入力します。**各URLの末尾に必ず / を付けるのを忘れないでください。**
- 環境変数を保存し、親アプリを再デプロイします。これによりCognitoの設定が更新され、SSOが有効になります。

![](https://assets.st-note.com/img/1769768049-J275Ta6rXCLFNMHApwl8dKjz.png?width=1200)

## まとめ

Amplify Gen2とCognito Hosted UIを組み合わせることで、既存のAmplify資産を活かしつつ、比較的少ないコードで堅牢なSSO環境を構築できることが分かりました。

特に、amplify/backend.ts でインフラ設定をコードとして管理できるAmplify Gen2のメリットは大きいと感じます。

また、昨年導入されたAmazon Cognitoのマネージドログイン機能により、ログインメニューのブランディングが可能になっています。

本稿では割愛しますが、マネジメントコンソールから設定することにより下記のようなログイン画面に簡単に差し替えることができます。

![](https://assets.st-note.com/img/1769768072-TUjCnQK6MdYlFpr0hwORPa2H.png?width=1200)

**詳細は下記ドキュメントをご確認ください。**

[https://docs.aws.amazon.com/cognito/latest/developerguide/managed-login-brandingeditor.html](https://docs.aws.amazon.com/cognito/latest/developerguide/managed-login-brandingeditor.html**)

**この記事が、Amplifyで複数アプリを管理する開発者の一助となれば幸いです。**
