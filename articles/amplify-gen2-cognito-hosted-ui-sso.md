---
title: "2025-07-29 Amplify Gen2とCognito Hosted UIで実現する！アプリ間SSOガイド"
emoji: "🔐"
type: "tech"
topics: ["AWS", "Amplify", "Cognito", "SSO", "React", "TypeScript"]
published: false
published_at: 2025-07-29
---

## はじめに
複数のAmplify Gen2業務アプリを横断して利用する社内ポータルが求められる中、ユーザーが同一人物であることを認識できるSSOが必要になりました。SAMLやOIDCをゼロから組み立てるとお世辞にもシンプルとは言えないため、Cognito Hosted UIを親アプリ(CognitoのAuth Provider)につなぎ、子アプリが同じ認証基盤を参照するかたちでSSOを実現します。

## 成果物
一方のアプリでログインすると別のアプリ側でも再ログインなしに同一ユーザーと認識されるという状態がゴールです。Amplify Gen2のホスティングとCognito Hosted UIの組み合わせなので、認証のロジック自体は最小限に抑えられます。

![親アプリのHosted UIにログイン](https://assets.st-note.com/img/1769767724-p2DdxjPOEheBFYyZ0TWgrMKA.png?width=1200)
![子アプリがトークンを受け取る](https://assets.st-note.com/img/1769767736-EUpOnjW3X5DPVQqLKvboAaxI.png?width=1200)
![リダイレクトせずに同一ユーザーを再利用](https://assets.st-note.com/img/1769767748-yQSkVm8DciMx92hpgfAlL1dH.png?width=1200)

## システム構成
親アプリがAuth Providerであり、Amplify Gen2の `cognito` リソースを生成して Hosted UI を提供します。子アプリは `referenceAuth` を利用して親アプリのユーザープールを参照し、Hosted UIトークンでログイン状態を再構築します。SSOの流れは図の通りです。

![親(Provider)と子(Consumer)の構成図](https://assets.st-note.com/img/1769767772-6zalTUWjN5CseInwXOQH0Fvu.png?width=1200)

## 前提条件
- Node.js/npm がインストールされていること
- AWSアカウントとAmplify Gen2アプリのデプロイ済み環境
- React(Vite)＋TypeScriptの知識
- [Amplify React クイックスタート](https://docs.amplify.aws/react/start/quickstart/)の手順を一度完了していること

## 手順

### 1. 親アプリ (Auth Provider) 設定

#### `package.json`
```json
{
  "dependencies": {
    "aws-amplify": "^7.1.1",
    "@aws-amplify/ui-react": "^4.0.5",
    "react": "18.3.0",
    "react-dom": "18.3.0"
  }
}
```

#### `amplify/auth/resource.ts`
```ts
import { defineAuthResource } from "@aws-amplify/cli-extensibility-helper";

export default defineAuthResource({
  name: "cognitoHosted",
  providerPlugin: "awscloudformation",
  authSelections: "identityPoolOnly",
  resourceName: "gen2-cognito-hosted",
  output: {
    CustomDomain: "portal-hosted-ui",
    UserPoolName: "amplify-gen2-portal-user-pool"
  }
});
```

#### `amplify/backend.ts`
```ts
import { defineAmplifyBackend } from "@aws-amplify/cli-extensibility-helper";

const TARGET_DOMAIN = process.env.TARGET_DOMAIN ?? "https://portal.example.com";

export default defineAmplifyBackend({
  auth: {
    name: "portalAuth",
    cognitoDomain: "portal-hosted-ui",
    callbackUrls: [`${TARGET_DOMAIN}/auth/callback`],
    logoutUrls: [`${TARGET_DOMAIN}/auth/logout`],
    allowedOAuthScopes: ["email", "openid", "profile"],
    allowedOAuthFlows: ["code"],
    custom: {
      outputs: {
        cognitoDomain: "portal-hosted-ui",
        targetDomain: TARGET_DOMAIN
      }
    }
  }
});
```

#### 補足
Cognitoドメインを使っている限り、バックエンドを `amplify backend remove` する前にコンソールからドメイン設定を削除しないとエラーになります。ドメイン残存のまま削除するとスタックに失敗するので先に削除してください。

#### `src/App.tsx`
```tsx
import { Auth } from "aws-amplify";
import outputs from "./amplify-outputs.json";

const cognitoDomain = outputs.custom?.cognitoDomain ?? `${window.location.origin}/oauth`;

Auth.configure({
  region: outputs.auth.region,
  userPoolId: outputs.auth.userPoolId,
  userPoolWebClientId: outputs.auth.userPoolClientId,
  oauth: {
    domain: cognitoDomain,
    scope: ["email", "openid", "profile"],
    redirectSignIn: outputs.auth.callbackUrls[0],
    redirectSignOut: outputs.auth.logoutUrls[0],
    responseType: "code"
  }
});

export const getCurrentUser = async () => {
  try {
    return await Auth.currentAuthenticatedUser();
  } catch (error) {
    console.error("AuthProvider: current user fetch failed", error);
    return null;
  }
};

export const signInWithRedirect = () => {
  Auth.federatedSignIn({ customProvider: "Cognito" });
};

export const signOut = async () => {
  try {
    await Auth.signOut();
  } catch (error) {
    console.error("AuthProvider: sign out failed", error);
  }
};
```

### 2. 子アプリ (Auth Consumer) 設定

#### `package.json`
```json
{
  "dependencies": {
    "aws-amplify": "^7.1.1",
    "@aws-amplify/ui-react": "^4.0.5",
    "react": "18.3.0",
    "react-dom": "18.3.0"
  }
}
```

#### `amplify/auth/resource.ts`
```ts
import { defineAuthResource, referenceAuth } from "@aws-amplify/cli-extensibility-helper";

export default defineAuthResource({
  name: "consumerAuth",
  providerPlugin: "awscloudformation",
  referenceAuth: referenceAuth({
    name: "gen2-cognito-hosted",
    userPoolId: process.env.REACT_APP_USER_POOL_ID ?? "",
    userPoolClientId: process.env.REACT_APP_USER_POOL_CLIENT_ID ?? ""
  })
});
```

#### `src/App.tsx`
```tsx
import { Auth } from "aws-amplify";

Auth.configure({
  region: process.env.REACT_APP_REGION ?? "ap-northeast-1",
  userPoolId: process.env.REACT_APP_USER_POOL_ID,
  userPoolWebClientId: process.env.REACT_APP_USER_POOL_CLIENT_ID,
  oauth: {
    domain: process.env.REACT_APP_COGNITO_DOMAIN ?? "",
    scope: ["email", "openid", "profile"],
    redirectSignIn: window.location.origin,
    redirectSignOut: window.location.origin,
    responseType: "code"
  }
});

export const fetchSession = async () => {
  try {
    return await Auth.currentSession();
  } catch (error) {
    console.error("AuthConsumer: session fetch failed", error);
    return null;
  }
};
```

### 3. Amplifyホスティングとドメイン設定
1. 親アプリの `amplify-outputs.json` から `user_pool_id` / `user_pool_client_id` / `identity_pool_id` を取得しておきます。IAMロールARNはAWSコンソールの `Roles` 画面からコピーします。
2. 子アプリのAmplify環境変数に `REACT_APP_USER_POOL_ID` / `REACT_APP_USER_POOL_CLIENT_ID` / `REACT_APP_REGION` / `REACT_APP_COGNITO_DOMAIN` を追加し、Hosted UIのドメインと一致させます。

![子アプリの環境変数設定](https://assets.st-note.com/img/1769767991-J8Wj5oUntPwC7ASlZrTFR6a3.png?width=1200)

3. 親アプリの `TARGET_DOMAIN` には連携先の子アプリURLをセットし、リダイレクトを通す仕組みにします。Amplify Consoleの環境変数で `TARGET_DOMAIN` を編集後、デプロイし直すと `amplify/backend.ts` で使えるようになります。

![親アプリのTARGET_DOMAIN設定①](https://assets.st-note.com/img/1769768023-40r2eOIPfAnS65L3iX1BZ8az.png?width=1200)
![親アプリのTARGET_DOMAIN設定②](https://assets.st-note.com/img/1769768049-J275Ta6rXCLFNMHApwl8dKjz.png?width=1200)

## まとめ
Amplify Gen2とCognito Hosted UIを組み合わせれば、比較的少ないコードでアプリ間SSOを構築できます。マネージドログインのブランディング機能を使えば認証画面の体験も整えられます。

![Managed Login Branding](https://assets.st-note.com/img/1769768072-TUjCnQK6MdYlFpr0hwORPa2H.png?width=1200)

詳細は [AWS Cognito Managed Login Branding](https://docs.aws.amazon.com/cognito/latest/developerguide/managed-login-brandingeditor.html) を参照してください。
