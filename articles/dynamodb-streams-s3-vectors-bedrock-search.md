---
title: "DynamoDB Streams × S3 Vectors × Bedrock で作る、準リアルタイム・セマンティック検索基盤"
emoji: "🔎"
type: "tech"
topics: ["aws", "amplify", "dynamodb", "bedrock", "vectorsearch"]
published: false
publication_name: "trust"
published_at: "2026-01-15"
---

こんにちは。エンジニアの岩波です。

この記事では、**AWS Amplify Gen2** と **Amazon Bedrock AgentCore** を活用し、**DynamoDB Streams** と **S3 Vectors** を組み合わせた、**「サーバーレスかつ待機コストゼロ」** のセマンティック検索システムの構築方法を解説します。

TypeScriptで定義するバックエンド構成から、AgentCore による自然言語でのデータ操作、Bedrock Knowledge Bases と連携した検索精度の実際まで、動くコードとデモをベースに詳しく紹介していきます。

**【想定読者】**

- DynamoDBのスケーラビリティは気に入っているが、検索機能（部分一致など）に課題を感じているエンジニア
- サーバーレスで手軽にRAG（Retrieval-Augmented Generation）やセマンティック検索を導入したい方
- Amazon S3 Vectors の具体的なアーキテクチャやコスト感を知りたい方

> **⚠️ 注意事項**
> 本記事で紹介するコード例は**概念的な実装例**であり、そのまま本番環境で使用できる完全なコードではありません。また、手元で動作したコードをブログ用に簡略化・修正しているため、実際のプロジェクトでは適切なエラーハンドリング、ログ出力、セキュリティ対策などを追加する必要があります。

---

## 背景: DynamoDBの検索における課題

本システムのデータストアである **DynamoDB** は、スケーラビリティとパフォーマンスに優れていますが、**「柔軟な検索」** という点ではいくつかの弱点があります。

1. **部分一致検索が苦手**: BeginsWith（前方一致）は高速ですが、Contains（部分一致）等のフィルタはテーブルスキャンとなり、データ量が増えるとパフォーマンスが劣化します。
2. **検索パターンの固定化**: 特定の属性で効率的に検索するには、事前に **GSI（グローバルセカンダリインデックス）** の設計と設定が必須であり、後からの要件変更に弱いです。
3. **意味検索の欠如**: 「〜のようなタスク」といった、キーワードが完全一致しない自然言語による検索は不可能です。

これらの課題を解決し、**「事前のインデックス設計なしで、曖昧な条件でも検索できる」** 仕組みを実現するために採用したのが、**Amazon S3 Vectors** です。

---

## Amazon S3 Vectors とは？

**Amazon S3 Vectors** は、2025年12月2日に一般提供（GA）が開始された、**S3ネイティブのベクトル検索機能**です。

### 1. 「専用ベクトルデータベース」との違い

従来、ベクトル検索を行うには **Pinecone, Weaviate, Amazon OpenSearch Service** といった「専用ベクトルデータベース」が必要でした。
これらはミリ秒単位の超高速レスポンスを誇りますが、以下のような課題がありました。

- **アイドルコスト**: 検索していない夜間でもサーバー代（または最低料金）がかかり続ける。
- **管理コスト**: クラスタのプロビジョニング、スケーリング、バージョンの管理が必要。

S3 Vectors は、**「S3に置くだけ」**でこれを解決します。検索リクエストが来た時だけ稼働するサーバーレス型のため、**待機コストはゼロ**です。

### 2. 対応ファイル形式（Bedrock Knowledge Bases連携）

本システムのように **Amazon Bedrock Knowledge Bases** と組み合わせることで、以下の一般的なドキュメント形式をそのまま検索対象にできます。

- **オフィス文書**: PDF, Word (.doc/docx), Excel (.xls/xlsx)
- **テキスト**: Plain Text (.txt), Markdown (.md), HTML, CSV

ユーザーはただファイルをS3にアップロードするだけで、裏側で自動的にベクトル化（float32形式への変換）が行われ、検索可能になります。

### 3. 料金と具体例（2026年1月時点）

最大の特徴は**「使った分だけ」**の従量課金です。

**項目料金目安備考ストレージ料金約 $0.06 / GB・月**データを保存している量に対してのみ課金**リクエスト料金約 $0.20 / GB**データの登録（PUT）時の一回払い**クエリ料金$0.0025 / 1,000回**実際に検索した回数分だけ

**【試算例】テキストファイル 1,000件 を運用した場合**

- 前提: 1ファイルあたりA4用紙1枚程度（数KB）、1536次元ベクトル (Titan Embeddings v1), us-east-1リージョン
- 総データ量: 約 10MB（ベクトル+メタデータ含む）

**月額ランニングコスト: $0.001 未満（約 0.15円）**

- もし専用DB（OpenSearch Serverless等）を使った場合: 最低でも **月額 $200〜** 程度が必要になるケースが多いです。
- S3 Vectors なら、小規模スタートや、検索頻度がまばらな社内ツールに最適です。

---

## 概要

このシステムでは、DynamoDBのTodoデータを自動的にベクトル化し、Amazon Bedrock Knowledge Baseで**セマンティック検索**を可能にしています。

これにより、以下のような自然言語での操作が実現されています:

- 「先週追加したタスクは？」
- 「優先度が高い未完了タスクを全部完了にして」
- 「明日までに終わらせるべきタスクを教えて」

### デモ動画

https://youtu.be/NygC2Z0lvLk

デモ動画では、以下の2つのパターンで意図通りの検索ができる様子を紹介しています。

**パターン1: 文脈によるグルーピング（「事務的なタスク」）**

> **入力:** 「開発以外の事務的なタスクある？」
> **結果:** 面接、オフィス移転、コスト削減（ビジネス寄り）がヒット
> **ポイント:** 「事務」という単語が含まれていなくても、バグ修正や勉強会（技術）を除外し、文脈的に合致するタスクのみを抽出しています。

**パターン2: 優先度と緊急度の複合判断**

> **入力:** 「今すぐやるべき重要なこと全部教えて」
> **結果:** コスト削減、バグ修正、面接（すべて優先度「高」）がヒット
> **ポイント:** 「今すぐやるべき重要」という抽象的な指示から、「優先度: 高」かつ「期限が近い」ものをAIが判断してリストアップしています。

---

### Amplify Gen2 プロジェクト構成

本システムは **AWS Amplify Gen2** で構築されており、以下のようなフォルダ構成になっています。

```
project/
├── amplify/
│   ├── backend.ts                    # バックエンド全体の定義
│   ├── data/
│   │   └── resource.ts               # DynamoDB (Todo) スキーマ定義
│   ├── functions/
│   │   └── sync-todo/
│   │       └── handler.ts            # DynamoDB Streams → S3 同期Lambda
│   ├── bedrock/
│   │   └── resource.ts               # Bedrock Knowledge Base 設定
│   ├── s3vectors/
│   │   └── resource.ts               # S3 Vector Index 定義
│   └── agentcore/
│       └── tools/
│           ├── knowledge-base-tool/  # 検索ツール
│           ├── todo-update-tool/     # 更新ツール
│           └── todo-delete-tool/     # 削除ツール
└── src/                              # フロントエンド (React等)
```

Amplify Gen2では、TypeScriptでバックエンドリソースを定義し、amplify/backend.ts で統合します。これにより、インフラとアプリケーションコードを一元管理できます。

**準リアルタイム反映:** Todo変更から数秒〜数十秒程度でKnowledge Baseに同期され、検索可能になります。

## データフロー全体像

```
1. DynamoDB Todoテーブルに変更（INSERT/UPDATE/DELETE）
     ↓
2. DynamoDB Streams が変更をキャプチャ
     ↓
3. sync-todo Lambda が起動
     ↓
4. S3バケットに保存:
   - kb-docs/todo-{id}.txt（テキストドキュメント）
   - kb-docs/todo-{id}.txt.metadata.json（メタデータ）
     ↓
5. StartIngestionJob を実行（Bedrock Knowledge Base）
     ↓
6. Bedrock が S3 ドキュメントを読み取り
     ↓
7. Titan Embeddings v1 でベクトル化（1536次元）
     ↓
8. PutVectors API で S3 Vector Index に保存
     ↓
9. Knowledge Base 検索が可能に（RetrieveCommand）
```

### フロー図

![](https://assets.st-note.com/img/1768382258-WDt9bfhnALIiY2zpZVJlge6q.png?width=1200)

## 実装詳細

### 1. DynamoDB Streams連携

**場所:** amplify/backend.ts

```
const todoTable = backend.data.resources.tables["Todo"];
syncLambda.addEventSource(
  new eventsources.DynamoEventSource(todoTable, {
    startingPosition: StartingPosition.LATEST,
    batchSize: 10,
    retryAttempts: 3,
  })
);
```

**仕組み:**

- Todoテーブルの変更（INSERT/UPDATE/DELETE）をキャプチャ

> LambdaにはDynamoDB Streamsへのアクセス権限や、Bedrock/S3を操作するための適切なIAM権限が必要です。
> 詳細は [**Amazon Bedrock の ID ベースのポリシー例**](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/security_iam_id-based-policy-examples.html) 等の公式ドキュメントを参照してください。

---

### 2. sync-todo Lambda処理

**場所:** amplify/functions/sync-todo/handler.ts

**ドキュメント生成**

テキストファイルで生成します。

```
const document = `Title: Todo ${todoId}
Content: ${content}
Status: ${isDone}
Priority: ${priority}
DueDate: ${dueDate}
Created: ${createdAt}
Updated: ${updatedAt}

--- Full Content ---
${content}`;
```

**メタデータ生成**

jsonファイルで生成します。
※メタデータのみ例外的にjson形式です。

> **メタデータ**とは？
> ベクトルデータ（埋め込み表現）に紐づけて保存できる**キーと値のペア情報**です。
> 例えば「作成日」「カテゴリ」「ステータス」などの属性を付与することで、ベクトル検索の結果に対して「2025年以降のデータに限る」「ステータスが未完了のものだけ」といった条件付けが可能になります。
> **対応型**: String, Number, Boolean, List
> **制限**: 1ベクトルあたり2KBまで

```
const metadata = {
  metadataAttributes: {
    documentType: "todo",
    todoId: todoId,
    status: isDone,
    priority: priority,
    dueDate: dueDate,
    createdAt: createdAt,
    updatedAt: updatedAt,
  },
};
```

**S3への保存**

```
// テキストドキュメント
await s3Client.send(
  new PutObjectCommand({
    Bucket: bucketName,
    Key: `kb-docs/todo-${todoId}.txt`,
    Body: document,
    ContentType: "text/plain",
  })
);

// メタデータ
await s3Client.send(
  new PutObjectCommand({
    Bucket: bucketName,
    Key: `kb-docs/todo-${todoId}.txt.metadata.json`,
    Body: JSON.stringify(metadata),
    ContentType: "application/json",
  })
);
```

**Ingestion Job開始**

```
await bedrockClient.send(
  new StartIngestionJobCommand({
    knowledgeBaseId,
    dataSourceId,
  })
);
```

---

### 3. S3Vectors自動インデックス管理

**場所:** amplify/s3vectors/resource.ts

**Vector Indexの設定**

```
const vectorIndex = new cdk.CfnResource(this, "VectorIndex", {
  type: "AWS::S3Vectors::Index",
  properties: {
    VectorBucketArn: vectorStoreBucketArn,
    IndexName: "bedrock-knowledge-base-default-index",
    Dimension: 1536, // Titan Embeddings v1の次元数
    DataType: "float32",
    DistanceMetric: "cosine", // コサイン類似度
  },
});
```

**S3Vectorsの特徴:**

- **自動インデックス更新**: PutVectors API実行時に内部で自動的に最適化
- **明示的な同期API不要**: AWSが内部で複雑なインデックスアルゴリズムを管理
- **高速検索**: コサイン類似度による効率的なベクトル検索

---

### 4. Bedrock Knowledge Base設定

**場所:** amplify/bedrock/resource.ts

**埋め込みモデル**

```
embeddingModelArn: `arn:aws:bedrock:${region}::foundation-model/amazon.titan-embed-text-v1`;
```

**Titan Embeddings v1:**

- 次元数: 1536
- 日本語・英語対応
- コスト効率が高い

**S3Vectors統合**

```
storageConfiguration: {
  type: "S3_VECTORS",
  s3VectorsConfiguration: {
    indexArn: vectorStoreIndexArn,
  },
}
```

---

## 検索とデータ操作の組み合わせ

このシステムの**最大の特徴**は、Knowledge Base検索とCRUD操作ツールを組み合わせた自然言語操作です。
これを実現しているのが **Amazon Bedrock AgentCore** です。AgentCoreは、AI Agentの構築・運用に特化したサーバーレスプラットフォームで、以下のような機能を提供します。

- **AgentCore Runtime**: 低レイテンシでAgentを実行するサーバーレス環境
- **AgentCore Memory**: 短期・長期記憶によるコンテキスト保持
- **AgentCore Gateway**: Lambda関数やAPIをAgent用ツールに変換
- **AgentCore Identity**: 安全な認証・認可の管理

本システムでは、AgentCoreのRuntimeとGateway機能を使い、「検索結果から得たtodoIdを使って、TodoUpdateツールを自動的に呼び出す」という連鎖実行を実現しています。

> [AgentCoreの詳細については、**Amazon Bedrock AgentCore 公式ドキュメント** を参照してください。](https://aws.amazon.com/jp/bedrock/agentcore/)

### Knowledge Base Tool

**場所:** amplify/agentcore/tools/knowledge-base-tool/handler.ts

```
const command = new RetrieveCommand({
  knowledgeBaseId: env.KNOWLEDGE_BASE_ID,
  retrievalQuery: { text: query },
  retrievalConfiguration: {
    vectorSearchConfiguration: {
      numberOfResults: 5, // 上位5件を取得
    },
  },
});

const response = await client.send(command);
```

**検索結果の構造:**

```
{
  "results": [
    {
      "text": "Title: Todo abc123\nContent: 資料作成\n...",
      "source": "s3://bucket/kb-docs/todo-abc123.txt",
      "score": 0.95,
      "metadata": {
        "todoId": "abc123",
        "status": "未完了",
        "priority": "高",
        "dueDate": "2026-01-15"
      }
    }
  ]
}
```

実際に取得されるレスポンスの例: score (類似度) が付与されており、確信度を可視化できます

![](https://assets.st-note.com/img/1768383125-OYHoilZP3TGWJACwKDr95q47.png)

---

### データ操作ツールとの連鎖実行

AgentCoreは検索結果の metadata.todoId を使用して、他のツールを自動的に呼び出します。

**シナリオ1: 検索結果を一括更新**

**ユーザー入力:**

```
「優先度が高いタスクを全部完了にして」
```

**AgentCore実行フロー:**

```
1. Knowledge Base Tool を実行
   - query: "優先度が高いタスク"
   - 結果: [
       { todoId: "abc123", priority: "高", status: "未完了" },
       { todoId: "def456", priority: "高", status: "未完了" }
     ]

2. 検索結果から todoId を抽出

3. 各 todoId に対して TodoUpdate Tool を実行
   - TodoUpdate({ id: "abc123", isDone: true })
   - TodoUpdate({ id: "def456", isDone: true })

4. DynamoDB が更新される

5. 再び DynamoDB Streams → S3 → Knowledge Base へ同期
```

**シナリオ2: 検索結果を削除**

**ユーザー入力:**

```
「先月のタスクで完了したものは削除して」
```

**AgentCore実行フロー:**

```
1. Knowledge Base Tool を実行
   - query: "先月のタスク 完了"
   - 結果: [{ todoId: "xyz789", status: "完了" }]

2. TodoDelete Tool を実行
   - TodoDelete({ id: "xyz789" })

3. DynamoDB から削除

4. S3 からも削除（sync-todo Lambda）

5. Knowledge Base から削除される
```

---

## メタデータフィルタリング

Knowledge Baseは以下のメタデータでフィルタリング可能です:

- **todoId**: 一意識別子（例: "abc123"）
- **documentType**: ドキュメント種別（例: "todo"）
- **status**: ステータス（例: "未完了" / "完了"）
- **priority**: 優先度（例: "低" / "中" / "高"）
- **dueDate**: 期限日（例: "2026-01-15"）
- **createdAt**: 作成日時（ISO8601形式）
- **updatedAt**: 更新日時（ISO8601形式）

### 複数条件での検索例

**ユーザー入力:**

```
「優先度が高くて期限が今週中の未完了タスクは？」
```

**Knowledge Base検索:**

- query: "優先度が高い 期限が今週中 未完了"
- メタデータで自動的にフィルタリング
- セマンティック検索とメタデータフィルタの組み合わせ

---

## おわりに：メタデータがあれば DynamoDB は不要？

**いいえ、役割が異なります。**
「あいまい検索」には S3 Vectors が最強ですが、「IDを指定した高速な読み書き」や「トランザクション管理」には DynamoDB が必須です。

**S3 Vectors Metadata の特徴:**

- **得意なこと**: 「〜のような内容」の検索、メタデータによる絞り込み（完全一致可）
- **苦手なこと**: 本文のみを対象としたキーワード完全一致検索（全文検索エンジンのような使い方は不向き）
- **主な用途**: 検索インデックス、RAG

**DynamoDB の特徴:**

- **得意なこと**: ID指定でのミリ秒単位アクセス、厳密なキーワード完全一致検索
- **苦手なこと**: 「〜を含む」等の部分一致検索、表記ゆれへの対応
- **主な用途**: アプリのデータ実体

本システムでは、**「検索は S3 Vectors」「データ管理は DynamoDB」と適材適所で使い分けています。
なお、型番などの厳密な検索**を行いたい場合は、**その値をメタデータとして抽出して付与する**必要があります。そうすることで、S3 Vectorsのフィルタ機能（equalsオペレータ）を使って完全一致検索が可能になります。

---

## まとめ

このベクトル検索システムの特徴:

1. **準リアルタイム反映**: DynamoDB変更から数秒〜数十秒でKnowledge Baseに反映
2. **自動インデックス管理**: S3Vectorsが内部で最適化、手動同期不要
3. **セマンティック検索**: 自然言語での柔軟な検索
4. **データ操作との連鎖**: 検索結果を使ったCRUD操作の自動実行

これにより、「先週追加したタスクを完了にして」のような**自然言語操作**が実現されています。

## 参考ページ

- [https://aws.amazon.com/jp/s3/features/vectors/](https://aws.amazon.com/jp/s3/features/vectors/)
- [https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/s3-vectors-bedrock-kb.html](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/s3-vectors-bedrock-kb.html)
- [https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Streams.Lambda.html](https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Streams.Lambda.html)
