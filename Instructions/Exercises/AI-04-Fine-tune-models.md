---
lab:
  title: Azure Databricks と Microsoft Foundry を使用して大規模言語モデルを微調整する
  description: トークン数の検証、微調整ジョブの送信、進行状況の監視など、Azure Databricks と Microsoft Foundry を使ってカスタム データセットで GPT-4.1 モデルを微調整する実践的な経験を得ます。 カスタマイズした微調整済みモデルをデプロイし、基本モデルでは提供されない専門知識を必要とするドメイン固有タスクに対してチャット入力候補 API でそれを使う方法を学びます。
  duration: 60 minutes
  level: 400
  islab: true
  primarytopics:
    - Azure Databricks
    - Azure Portal
    - Microsoft Foundry
---

# Azure Databricks と Microsoft Foundry を使用して大規模言語モデルを微調整する

Azure Databricks と Microsoft Foundry を使用すると、独自のデータで大規模言語モデルを微調整して、ドメイン固有のパフォーマンスを向上させることができます。 このラボでは、Azure Databricks は、データの準備、Microsoft Foundry の微調整 API の呼び出し、結果のモデルのテストを行う開発環境として機能します。 微調整を行うと、比較的小さな精選されたデータセットを使用して、事前トレーニング済みの基本モデルを特殊なタスクに適応できます。

このラボは完了するまで、約 **60** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Microsoft Foundry リソースとプロジェクトを作成する

まだ持っていない場合は、Azure サブスクリプションで Microsoft Foundry リソースとプロジェクトを作成します。

> **注**: Foundry リソースを作成するために必要なものは、サブスクリプション、リソース グループ、リージョン、名前のみです。 Key Vault または Application Insights リソースは必要ありません。

1. **Azure portal** (`https://portal.azure.com`) にサインインします。
2. 次のリンクを使用して Foundry リソースの作成ページを開きます: `https://portal.azure.com/#create/Microsoft.CognitiveServicesAIFoundry`
3. **[作成]** ページの **[基本]** タブで、次の情報を指定します。
    - **[サブスクリプション]**: *Azure サブスクリプションを選択します*
    - **[リソース グループ]**: *リソース グループを作成または選択します*
    - **[リージョン]**: *以下のいずれかのリージョンから**ランダム**に選択する*\*
        - 米国中北部
        - スウェーデン中部
    - **[名前]**: "*希望する一意の名前*"
4. **[確認と作成]**、**[作成]** の順に選択し、デプロイが完了するまで待ちます。

> \* Foundry リソースは、リージョンのクォータによって制限されます。 一覧表示されているリージョンには、この演習で使用されるモデル タイプの既定のクォータが含まれています。 リージョンをランダムに選択することで、サブスクリプションを他のユーザーと共有しているシナリオで、1 つのリージョンがクォータ制限に達するリスクが軽減されます。 演習の後半でクォータ制限に達した場合は、別のリージョンに別のリソースを作成する必要が生じる可能性があります。

5. デプロイが完了したら、デプロイされたリソースに移動します。 左側のペインの **[リソース管理]** で **[キーとエンドポイント]** を選択し、**[エンドポイント]** をコピーします。これはこの演習で後ほど使用します。

6. **[概要]** ページで、**[Microsoft Foundry に移動]** を選択してリソースを Foundry ポータルで開きます (または、`https://ai.azure.com` に直接移動します)。

7. **Microsoft Foundry** で、Foundry リソース内に新しい**プロジェクト**を作成します。
    - 左上隅にあるプロジェクト名を選択して **[新しいプロジェクトの作成]** を選択します。
    - **プロジェクト名**を入力し、**[プロジェクトの作成]** を選択します。
    - プロジェクトが作成されるまで待ちます。

8. Cloud Shell を起動し、次の 2 つのコマンドを実行して API テスト用の一時的な認証トークンを取得します。 以前にコピーしたエンドポイントと共に保持します。

    ```bash
    az account get-access-token --resource https://cognitiveservices.azure.com
    az account get-access-token --resource https://management.azure.com
    ```

    >**注**: 最初のトークンは OpenAI/Cognitive Services API 呼び出しに使用されます。2 つ目は Azure 管理 API (モデル デプロイ) に使用されます。 コピーする必要があるのはそれぞれの `accessToken` フィールド値だけであり、JSON 出力全体では**ありません**。

## 必要なモデルをデプロイする

Microsoft Foundry を使用すると、モデルのデプロイ、管理、探索を行うことができます。 基本モデルをデプロイし、それを後で微調整します。

> **注**: Microsoft Foundry を使う際、実行するタスクを提案するメッセージ ボックスが表示される場合があります。 これらを閉じて、この演習の手順に従うことができます。

1. **Microsoft Foundry** の左側のペインで、**[モデル + エンドポイント]** ページを選択して既存のモデル デプロイを表示します。 まだない場合は、次の設定で **gpt-4.1** モデルの新しいデプロイを作成します。
    - **デプロイ名**: *gpt-4.1*
    - **デプロイの種類**:Standard
    - **モデル バージョン**: *2025-04-14*
    - **1 分あたりのトークン数のレート制限**:10K\*
    - **コンテンツ フィルター**: 既定
    - **動的クォータを有効にする**: 無効
    
> \* この演習は、1 分あたり 10,000 トークンのレート制限内で余裕を持って完了できます。またこの制限によって、同じサブスクリプションを使用する他のユーザーのために容量を残すこともできます。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

1. **Azure portal** (`https://portal.azure.com`) にサインインします。
2. 次の設定で **Azure Databricks** リソースを作成します。
    - **サブスクリプション**: Microsoft Foundry リソースの作成に使用したのと同じ Azure サブスクリプションを選択します**
    - **リソース グループ**: Microsoft Foundry リソースを作成したリソース グループと同じです**
    - **リージョン**: Microsoft Foundry リソースを作成したリージョンと同じです**
    - **[名前]**: "*希望する一意の名前*"
    - **価格レベル**: *Premium*
    - **ワークスペースの種類**: *ハイブリッド*
    - **マネージド リソース グループ**: 既定値のままにするか、一意の名前を入力します**

3. **[確認および作成]** を選択し、デプロイが完了するまで待ちます。 次にリソースに移動し、ワークスペースを起動します。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 17.3 LTS **<u>ML</u>** 以降のランタイム バージョンを備えたクラスターが既にある場合は、この手順をスキップし、そのクラスターを使用してこの演習を完了できます。

1. Azure portal で、Azure Databricks ワークスペースが作成されたリソース グループを参照します。
2. Azure Databricks サービス リソースを選択します。
3. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

> **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

4. 左側のサイドバーで、**[(+) 新規]** タスクを選択し、**[クラスター]** を選択します。
5. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
    - **クラスター名**: "ユーザー名の" クラスター (既定のクラスター名)**
    - **ポリシー**:Unrestricted
    - **機械学習**: 有効
    - **Databricks Runtime**: 17.3 LTS
    - **Photon Acceleration を使用する**: <u>オフ</u>にする
    - **ワーカー タイプ**:Standard_D4ds_v5
    - **シングル ノード**:オン
    - **終了までの時間**: 30 分間の非アクティブ状態

6. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。

## 新しいノートブックの作成とデータの取り込み

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。 **[接続]** ドロップダウン リストで、まだ選択されていない場合はクラスターを選択します。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

1. ノートブックの最初のセルで、次の SQL クエリを入力して、この演習のデータを既定のカタログ内に格納するために使用する新しいボリュームを作成します。

    ```python
   %sql 
   CREATE VOLUME <catalog_name>.default.fine_tuning;
    ```

1. `<catalog_name>` を既定のカタログの名前に置き換えます。 サイドバーで **[カタログ]** を選択すると、その名前を確認できます。
1. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。
1. 新しいセルで、次のコードを実行して、"シェル" コマンドを使用して GitHub から Unity カタログにデータをダウンロードします。**

    ```python
   %sh
   wget -O /Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/training_set.jsonl
   wget -O /Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/validation_set.jsonl
    ```

3. 新しいセルで、この演習の冒頭でコピーしたアクセス情報を含む次のコードを実行して、Microsoft Foundry リソースを使用するときの認証用に永続的な環境変数を割り当てます。

    ```python
   import os

   os.environ["AZURE_OPENAI_ENDPOINT"] = "your_foundry_endpoint"
   os.environ["COGNITIVE_SERVICES_TOKEN"] = "your_cognitiveservices_access_token"  # from: az account get-access-token --resource https://cognitiveservices.azure.com
   os.environ["MANAGEMENT_TOKEN"] = "your_management_access_token"                # from: az account get-access-token --resource https://management.azure.com
    ```
     
## トークン数を検証する

`training_set.jsonl` と `validation_set.jsonl` はどちらも、微調整されたモデルをトレーニングおよび検証するためのデータ ポイントとして機能する、`user` と `assistant` の間でのさまざまな会話の例で構成されています。 この演習のデータセットは小さいと考えられますが、より大きなデータセットを使用する場合は、トークンを単位として、LLM のコンテキストの長さが最大であることに留意することが重要です。 そのため、モデルをトレーニングする前にデータセットのトークン数を確認し、必要に応じて変更することができます。 

1. 新しいセルで、次のコードを実行して、各ファイルのトークン数を検証します。

    ```python
   import json
   import tiktoken
   import numpy as np
   from collections import defaultdict

   encoding = tiktoken.get_encoding("cl100k_base")

   def num_tokens_from_messages(messages, tokens_per_message=3, tokens_per_name=1):
       num_tokens = 0
       for message in messages:
           num_tokens += tokens_per_message
           for key, value in message.items():
               num_tokens += len(encoding.encode(value))
               if key == "name":
                   num_tokens += tokens_per_name
       num_tokens += 3
       return num_tokens

   def num_assistant_tokens_from_messages(messages):
       num_tokens = 0
       for message in messages:
           if message["role"] == "assistant":
               num_tokens += len(encoding.encode(message["content"]))
       return num_tokens

   def print_distribution(values, name):
       print(f"\n##### Distribution of {name}:")
       print(f"min / max: {min(values)}, {max(values)}")
       print(f"mean / median: {np.mean(values)}, {np.median(values)}")

   files = ['/Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl', '/Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl']

   for file in files:
       print(f"File: {file}")
       with open(file, 'r', encoding='utf-8') as f:
           dataset = [json.loads(line) for line in f]

       total_tokens = []
       assistant_tokens = []

       for ex in dataset:
           messages = ex.get("messages", {})
           total_tokens.append(num_tokens_from_messages(messages))
           assistant_tokens.append(num_assistant_tokens_from_messages(messages))

       print_distribution(total_tokens, "total tokens")
       print_distribution(assistant_tokens, "assistant tokens")
       print('*' * 75)
    ```

参考として、この演習で使用されるモデル GPT-4.1 のコンテキスト制限は 1,047,576 トークンです (ただし、標準デプロイは 300,000 トークンに制限されます)。

## 微調整ファイルを Microsoft Foundry にアップロードする

モデルの微調整を開始する前に、OpenAI クライアントを初期化し、その環境に微調整ファイルを追加して、ジョブの初期化に使用されるファイル ID を生成する必要があります。

> **重要**: Azure Databricks でコードを実行する場合、`DefaultAzureCredential` は、サインインしているユーザー アカウントではなく、**Databricks ワークスペースのマネージド ID** として認証されます。 つまり、**Cognitive Services OpenAI 共同作成者**ロールを自分のアカウントに割り当てるだけでは不十分です。マネージド ID にもロールが必要です。または、ユーザー トークンを直接使用して認証する必要があります。 次のコードでは、先ほど Cloud Shell を使用して取得した `TEMP_AUTH_TOKEN` と共に `azure_ad_token` を使用します。これは独自の ID として実行され、この問題が回避されます。

1. 新しいセルで次のコードを実行します。

     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      azure_ad_token = os.getenv("COGNITIVE_SERVICES_TOKEN"),
      api_version = "2025-04-01-preview"  # This API version or later is required to access seed/events/checkpoint features
    )

    training_file_name = '/Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl'
    validation_file_name = '/Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl'

    training_response = client.files.create(
        file = open(training_file_name, "rb"), purpose="fine-tune"
    )
    training_file_id = training_response.id

    validation_response = client.files.create(
        file = open(validation_file_name, "rb"), purpose="fine-tune"
    )
    validation_file_id = validation_response.id

    print("Training file ID:", training_file_id)
    print("Validation file ID:", validation_file_id)
     ```

## 微調整ジョブを送信する

微調整ファイルが正常にアップロードされたので、微調整トレーニング ジョブを送信できるようになりました。 トレーニングが完了するまでに 1 時間以上かかるのは珍しいことではありません。 トレーニングを終えたら、左側のペインで **[微調整]** オプションを選んで、Microsoft Foundry で結果を確認できます。

1. 新しいセルで、次のコードを実行して、微調整トレーニング ジョブを開始します。

    ```python
   response = client.fine_tuning.jobs.create(
       training_file = training_file_id,
       validation_file = validation_file_id,
       model = "gpt-4.1-2025-04-14",
       seed = 105 # seed parameter controls reproducibility of the fine-tuning job. If no seed is specified one will be generated automatically.
   )

   job_id = response.id
    ```

`seed` パラメーターは、微調整ジョブの再現性を制御します。 同じシードおよびジョブ パラメーターを渡すと同じ結果が得られますが、まれに異なる場合があります。 シードが指定されていない場合は、自動的に生成されます。

2. 新しいセルで、次のコードを実行して、微調整ジョブの状態を監視できます。

    ```python
   print("Job ID:", response.id)
   print("Status:", response.status)
    ```

>**注**: 左側のサイド バーで **[微調整]** を選択しても、Microsoft Foundry のジョブの状態を監視できます。

3. ジョブの状態が `succeeded` に変わったら、次のコードを実行して、最終的な結果を取得します。

    ```python
   response = client.fine_tuning.jobs.retrieve(job_id)

   print(response.model_dump_json(indent=2))
   fine_tuned_model = response.fine_tuned_model
    ```
   
## 微調整されたモデルをデプロイする

これで微調整済みモデルができたので、カスタマイズされたモデルとしてそれをデプロイし、Microsoft Foundry の**チャット** プレイグラウンド、またはチャット入力候補 API のいずれかを使って、他のデプロイ済みモデルと同様に使用できます。

1. 新しいセルで、次のコードを実行して、微調整されたモデルをデプロイします。
   
    ```python
   import json
   import requests

   token = os.getenv("MANAGEMENT_TOKEN")
   subscription = "<YOUR_SUBSCRIPTION_ID>"
   resource_group = "<YOUR_RESOURCE_GROUP_NAME>"
   resource_name = "<YOUR_AZURE_AI_SERVICES_RESOURCE_NAME>"  # The Azure AI Services resource name from your AI Foundry project settings
   model_deployment_name = "gpt-4.1-ft"

   deploy_params = {'api-version': "2023-05-01"}
   deploy_headers = {'Authorization': 'Bearer {}'.format(token), 'Content-Type': 'application/json'}

   deploy_data = {
       "sku": {"name": "standard", "capacity": 1},
       "properties": {
           "model": {
               "format": "OpenAI",
               "name": fine_tuned_model
           }
       }
   }
   deploy_data = json.dumps(deploy_data)

   request_url = f'https://management.azure.com/subscriptions/{subscription}/resourceGroups/{resource_group}/providers/Microsoft.CognitiveServices/accounts/{resource_name}/deployments/{model_deployment_name}'

   print('Creating a new deployment...')

   r = requests.put(request_url, params=deploy_params, headers=deploy_headers, data=deploy_data)

   print(r)
   print(r.reason)
   print(r.json())
    ```

2. 新しいセルで、次のコードを実行して、チャット入力候補呼び出しでカスタマイズしたモデルを使用します。
   
    ```python
   import os
   from openai import AzureOpenAI

   client = AzureOpenAI(
     azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
     azure_ad_token = os.getenv("COGNITIVE_SERVICES_TOKEN"),
     api_version = "2024-02-01"
   )

   response = client.chat.completions.create(
       model = "gpt-4.1-ft", # model = "Custom deployment name you chose for your fine-tuning model"
       messages = [
           {"role": "system", "content": "You are a helpful assistant."},
           {"role": "user", "content": "Does Microsoft Foundry support customer managed keys?"},
           {"role": "assistant", "content": "Yes, customer managed keys are supported by Microsoft Foundry."},
           {"role": "user", "content": "Do other Azure AI services support this too?"}
       ]
   )

   print(response.choices[0].message.content)
    ```
 
## クリーンアップ

完了したら、`https://ai.azure.com` で **Microsoft Foundry** 内のデプロイまたは Microsoft Foundry プロジェクト全体を必ず削除してください。

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。