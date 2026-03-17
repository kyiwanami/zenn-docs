---
title: "DynamoDB Streams × S3 Vectors × Bedrock で作る、準リアルタイム・セマンティック検索基盤"
emoji: "🔎"
type: "tech"
topics: ["aws", "amplify", "dynamodb", "bedrock", "vectorsearch"]
published: false
published_at: "2026-01-15"
---

## 導入

Amplify Gen2 と Amazon Bedrock AgentCore を活用し、DynamoDB Streams と S3 Vectors を組み合わせた「サーバーレスかつ待機コストゼロ」のセマンティック検索システムを構築しました。

この記事では、Todo データを例に、DynamoDB に保存されたレコードを変更契機で自動ベクトル化し、Amazon Bedrock Knowledge Bases と連携させることで、自然言語ベースの検索とデータ操作を両立する構成を解説します。

## 想定読者

- Amplify Gen2 でサーバーレスなアプリケーション基盤を構築している方
- DynamoDB のデータを自然言語で検索できるようにしたい方
- Amazon Bedrock AgentCore や Knowledge Bases を業務アプリに組み込みたい方

> 注意事項
>
> 本記事の内容は 2026 年 1 月時点の情報をもとにしています。Amazon S3 Vectors や Bedrock 関連機能は今後仕様や料金が変わる可能性があります。実運用前には必ず最新の公式ドキュメントをご確認ください。

## 背景: DynamoDB 検索の課題

DynamoDB は高性能な NoSQL データベースですが、アプリケーションによっては検索体験にいくつかの制約があります。

- 部分一致検索が難しい  
  `contains` を使ったフィルタはできますが、大量データでは効率が悪く、全文検索のような使い方には向いていません。
- 検索パターンが固定化しやすい  
  GSI やアクセスパターンを事前に設計する前提のため、後から自由検索を追加したくなると設計が窮屈になります。
- 意味検索ができない  
  「期限が近くて重要そうなタスク」や「似た内容の作業」など、文脈を理解した検索はそのままではできません。

この課題に対して、DynamoDB を正本データストアとして維持しつつ、検索用途だけをベクトル化して補完する構成が有効です。

## Amazon S3 Vectors とは？

Amazon S3 Vectors は、S3 上にベクトルデータを保存し、Amazon Bedrock Knowledge Bases などと組み合わせて検索活用できる仕組みです。

### 専用ベクトルDBとの違い

専用のベクトルデータベースを別途運用する場合、クラスターの維持、監視、スケーリング、待機コストなどを考慮する必要があります。

一方で S3 Vectors は、S3 をベースにした構成のため次のような利点があります。

- 常時稼働の専用クラスタを持たずに済む
- S3 ベースなので運用負荷が低い
- サーバーレス構成と相性が良く、待機コストを抑えやすい

### Bedrock Knowledge Bases 連携で扱えるファイル形式

Bedrock Knowledge Bases と連携して取り込めるファイル形式には、たとえば次のようなものがあります。

- `.txt`
- `.md`
- `.html`
- `.csv`
- `.pdf`
- `.docx`

アプリケーションデータをそのまま検索対象にするというより、検索しやすい文書形式に整形して S3 に配置するのが実装上のポイントです。

### 料金と具体例（2026年1月時点）

2026 年 1 月時点では、概算として次のようなイメージです。

- ストレージ: 約 `$0.06 / GB / 月`
- 登録: 約 `$0.20 / GB`
- クエリ: `$0.0025 / 1000 回`

たとえば 1,000 件の Todo ドキュメントを保存し、合計サイズが数 MB から数十 MB 程度の小規模運用であれば、ストレージコストはかなり小さく抑えられます。仮に 1,000 件分をまとめて 1 GB 相当としても、月額ストレージは約 `$0.06`、登録時は約 `$0.20`、検索 1,000 回で `$0.0025` 程度です。

この価格感は、まず小さく始めて検索体験を追加したいケースと非常に相性が良いです。

## 概要

今回の仕組みでは、Todo データを自動ベクトル化し、次のような自然言語での検索や操作を可能にします。

- 「明日までに終わらせるべきタスクを教えて」
- 「重要だけど未着手の作業を優先順で出して」
- 「会議準備に関係する Todo をまとめて表示して」
- 「急ぎで対応が必要なタスクを完了済みにして」
- 「買い物系のメモだけまとめて削除して」

単なるキーワード一致ではなく、説明文や文脈を含めた意味検索ができる点が大きな違いです。

## デモ動画説明

デモでは主に次の 2 パターンを確認できます。

### 1. 文脈によるグルーピング

たとえば「引っ越し準備に関する Todo」と聞くと、文言が完全一致していなくても、荷造り、住所変更、ライフライン手続きなど、意味的に近いタスクがまとまって取得されます。

### 2. 優先度・緊急度の複合判断

「今日中にやるべき重要タスク」のような曖昧な要求に対しても、期日、priority、status などのメタデータと、本文の意味検索を組み合わせて候補を絞り込めます。

## Amplify Gen2 プロジェクト構成

```text
amplify/
├── backend.ts
├── data/
│   └── resource.ts
├── functions/
│   └── sync-todo/
│       ├── handler.ts
│       └── resource.ts
├── s3vectors/
│   └── resource.ts
└── bedrock/
    └── resource.ts
```

## データフロー全体像

システム全体の流れは次のとおりです。

1. Amplify Gen2 で定義した Todo モデルのデータが DynamoDB に保存される
2. Todo の新規作成・更新・削除を DynamoDB Streams が検知する
3. Streams をトリガーに `sync-todo` Lambda が起動する
4. Lambda が DynamoDB レコードから検索用ドキュメント本文を生成する
5. 同時に絞り込み用メタデータを生成する
6. 生成したドキュメントを S3 に保存する
7. `StartIngestionJobCommand` で Bedrock Knowledge Base の取り込みを開始する
8. Knowledge Base が S3 Vectors にインデックスを作成する
9. Bedrock AgentCore から自然言語検索やデータ操作ツールを連鎖実行する

![データフロー全体像](https://assets.st-note.com/img/1768382258-WDt9bfhnALIiY2zpZVJlge6q.png?width=1200)

## 実装詳細

### 1. DynamoDB Streams連携 (`amplify/backend.ts`)

まず、Amplify Gen2 のバックエンド定義で `sync-todo` 関数をデータ変更と連携させます。

```ts
import { defineBackend } from "@aws-amplify/backend";
import { data } from "./data/resource";
import { syncTodo } from "./functions/sync-todo/resource";
import { bedrock } from "./bedrock/resource";
import { s3vectors } from "./s3vectors/resource";

const backend = defineBackend({
  data,
  syncTodo,
  bedrock,
  s3vectors,
});

backend.syncTodo.resources.lambda.addEventSourceMapping("TodoStreamMapping", {
  eventSourceArn: backend.data.resources.tables["Todo"].tableStreamArn,
  startingPosition: "LATEST",
  batchSize: 10,
  retryAttempts: 2,
});
```

この構成により、Todo テーブルの変更が Lambda に自動連携されます。

### 2. sync-todo Lambda処理 (`amplify/functions/sync-todo/handler.ts`)

`sync-todo` Lambda では、DynamoDB のレコードから検索用ドキュメントとメタデータを生成し、それを S3 に保存したうえで Knowledge Base の再取り込みを開始します。

#### ドキュメント生成コード

```ts
const documentBody = [
  `タイトル: ${todo.title}`,
  `説明: ${todo.description ?? ""}`,
  `ステータス: ${todo.status}`,
  `優先度: ${todo.priority}`,
  `期限: ${todo.dueDate ?? ""}`,
]
  .filter(Boolean)
  .join("\n");
```

#### メタデータ生成コード

```ts
const metadata = {
  todoId: todo.id,
  documentType: "todo",
  status: todo.status,
  priority: todo.priority,
  dueDate: todo.dueDate ?? "",
  createdAt: todo.createdAt,
  updatedAt: todo.updatedAt,
};
```

#### S3保存コード

```ts
await s3Client.send(
  new PutObjectCommand({
    Bucket: env.TODO_DOCUMENT_BUCKET_NAME,
    Key: `todos/${todo.id}.md`,
    Body: [
      "---",
      JSON.stringify(metadata),
      "---",
      "",
      documentBody,
    ].join("\n"),
    ContentType: "text/markdown",
  }),
);
```

#### `StartIngestionJobCommand` のコード

```ts
await bedrockAgentClient.send(
  new StartIngestionJobCommand({
    knowledgeBaseId: env.KNOWLEDGE_BASE_ID,
    dataSourceId: env.KNOWLEDGE_BASE_DATA_SOURCE_ID,
    description: `sync todo: ${todo.id}`,
  }),
);
```

この処理で、DynamoDB の更新から Knowledge Base への反映までを準リアルタイムでつなげられます。

### 3. S3Vectors自動インデックス管理 (`amplify/s3vectors/resource.ts`)

S3 Vectors 側のリソースは、CloudFormation ベースで `AWS::S3Vectors::Index` を定義します。

```ts
export const s3VectorsIndex = {
  Type: "AWS::S3Vectors::Index",
  Properties: {
    IndexName: "todo-semantic-search-index",
    DataType: "float32",
    Dimension: 1536,
    DistanceMetric: "cosine",
  },
};
```

この構成の特徴は次の 3 点です。

- S3 を中心にしたサーバーレス構成でベクトルインデックスを管理できる
- 埋め込み次元数や距離計算方法を明示して構成できる
- 専用ベクトル DB を常時起動せずに済み、待機コストを抑えやすい

### 4. Bedrock Knowledge Base設定 (`amplify/bedrock/resource.ts`)

Knowledge Base 側では、埋め込みモデルとストレージ設定を関連付けます。

```ts
export const knowledgeBase = {
  knowledgeBaseConfiguration: {
    type: "VECTOR",
    vectorKnowledgeBaseConfiguration: {
      embeddingModelArn:
        "arn:aws:bedrock:ap-northeast-1::foundation-model/amazon.titan-embed-text-v1",
    },
  },
  storageConfiguration: {
    type: "S3_VECTORS",
    s3VectorsConfiguration: {
      vectorIndexArn: "arn:aws:s3vectors:ap-northeast-1:123456789012:index/todo-semantic-search-index",
      indexName: "todo-semantic-search-index",
    },
  },
};
```

Titan Embeddings v1 の特徴としては、次のような点が扱いやすいです。

- テキスト埋め込み用途として使いやすい標準的なモデルである
- Bedrock エコシステム内で統一して扱いやすい
- 業務データの意味検索基盤を作るうえで十分な精度と運用性を持つ

## 検索とデータ操作の組み合わせ

ここで重要なのは、検索だけで終わらせず、AgentCore からデータ操作までつなげることです。

### Amazon Bedrock AgentCore の機能

- 自然言語入力の解釈
- Knowledge Base を使った関連情報の検索
- 外部ツールの呼び出し
- 検索結果を踏まえた連鎖実行

これにより、「探す」と「更新する」「削除する」を 1 つの会話体験にまとめられます。

### Knowledge Base Tool (`RetrieveCommand`) コード

Knowledge Base から候補を引く処理は次のようになります。

```ts
const response = await bedrockAgentRuntimeClient.send(
  new RetrieveCommand({
    knowledgeBaseId: env.KNOWLEDGE_BASE_ID,
    retrievalQuery: {
      text: "優先度が高く、まだ未完了のタスクを探して",
    },
  }),
);
```

### レスポンス例

```json
{
  "retrievalResults": [
    {
      "content": {
        "text": "タイトル: 請求書送付\n説明: 月末までに送付する\nステータス: IN_PROGRESS\n優先度: HIGH"
      },
      "location": {
        "s3Location": {
          "uri": "s3://todo-documents/todos/abc123.md"
        }
      },
      "metadata": {
        "todoId": "abc123",
        "status": "IN_PROGRESS",
        "priority": "HIGH"
      }
    }
  ]
}
```

![検索とデータ操作の連携](https://assets.st-note.com/img/1768383125-OYHoilZP3TGWJACwKDr95q47.png)

### データ操作ツールとの連鎖実行

検索結果を取得したあと、そのままデータ操作ツールへつなげられます。

#### 一括更新シナリオ

「今週中が期限で、優先度が高いタスクを全部 `IN_PROGRESS` にして」のような命令に対して、

1. Knowledge Base で候補を取得
2. 取得した `todoId` 群を抽出
3. 更新用ツールに渡して一括更新

という流れで処理できます。

#### 一括削除シナリオ

「買い物メモ系の古いタスクを全部削除して」のようなケースでは、

1. 意味検索で対象候補を抽出
2. 削除対象をユーザーに確認
3. 削除ツールで DynamoDB のデータを一括削除

といった連鎖実行が可能です。

## メタデータフィルタリング

今回の構成では、次のメタデータを付与しています。

- `todoId`
- `documentType`
- `status`
- `priority`
- `dueDate`
- `createdAt`
- `updatedAt`

これにより、意味検索だけでなく、構造化条件との併用がしやすくなります。

たとえば次のような複数条件検索ができます。

- `status = "IN_PROGRESS"`
- `priority = "HIGH"`
- `dueDate <= "2026-01-31"`

つまり、「意味的に近い候補を探す」と「明示的な業務条件で絞る」を両立できます。

## おわりに

今回の構成では、S3 Vectors Metadata と DynamoDB がそれぞれ別の役割を持ちます。

- DynamoDB は業務データの正本として最新状態を保持する
- S3 Vectors Metadata は意味検索しやすい形に整えた検索用インデックスとして使う

この役割分担により、更新系は DynamoDB に任せつつ、検索体験だけを Bedrock と S3 Vectors で強化できます。

## まとめ

- DynamoDB Streams により、データ変更をトリガーに自動同期できる
- S3 Vectors を使うことで、待機コストゼロに近い検索基盤を組みやすい
- Bedrock Knowledge Base と AgentCore により、自然言語検索と操作をつなげられる
- メタデータフィルタを組み合わせることで、実務に耐える検索精度を出しやすい

検索だけでなく、操作まで含めて自然言語化したい場合、この構成はかなり扱いやすい選択肢でした。

## 参考ページ

- [Amazon S3 Vectors](https://aws.amazon.com/jp/s3/features/vectors/)
- [Amazon S3 Vectors と Bedrock Knowledge Bases の連携](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/s3-vectors-bedrock-kb.html)
- [DynamoDB Streams と Lambda の連携](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Streams.Lambda.html)
