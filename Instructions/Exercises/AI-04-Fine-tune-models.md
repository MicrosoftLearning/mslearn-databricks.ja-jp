---
lab:
  title: Azure Databricks と Azure OpenAI を使用して大規模言語モデルを微調整する
---

# Azure Databricks と Azure OpenAI を使用して大規模言語モデルを微調整する

Azure Databricks を使用すると、ユーザーは独自のデータを使用して微調整することで、LLM の利点を特殊なタスクに活用できるようになり、ドメイン固有のパフォーマンスが向上します。 Azure Databricks を使用して言語モデルを微調整するために、モデルの完全な微調整のプロセスを簡略化する Mosaic AI モデル トレーニングのインターフェイスを利用できます。 この機能を使用すると、カスタム データを使用してモデルを微調整し、チェックポイントを MLflow に保存して、微調整したモデルを完全に制御できます。

このラボは完了するまで、約 **60** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure OpenAI リソースをプロビジョニングする

まだ持っていない場合は、Azure サブスクリプションで Azure OpenAI リソースをプロビジョニングします。

1. **Azure portal** (`https://portal.azure.com`) にサインインします。
2. 次の設定で **Azure OpenAI** リソースを作成します。
    - **[サブスクリプション]**: "Azure OpenAI Service へのアクセスが承認されている Azure サブスクリプションを選びます"**
    - **[リソース グループ]**: *リソース グループを作成または選択します*
    - **[リージョン]**: *以下のいずれかのリージョンから**ランダム**に選択する*\*
        - 米国東部 2
        - 米国中北部
        - スウェーデン中部
        - スイス西部
    - **[名前]**: "*希望する一意の名前*"
    - **価格レベル**: Standard S0

> \* Azure OpenAI リソースは、リージョンのクォータによって制限されます。 一覧表示されているリージョンには、この演習で使用されるモデル タイプの既定のクォータが含まれています。 リージョンをランダムに選択することで、サブスクリプションを他のユーザーと共有しているシナリオで、1 つのリージョンがクォータ制限に達するリスクが軽減されます。 演習の後半でクォータ制限に達した場合は、別のリージョンに別のリソースを作成する必要が生じる可能性があります。

3. デプロイが完了するまで待ちます。 次に、Azure portal でデプロイされた Azure OpenAI リソースに移動します。

4. 左側のペインで、**[リソース管理]** の下の **[キーとエンドポイント]** を選択します。

5. エンドポイントと使用可能なキーの 1 つをコピーしておきます。この演習で、後でこれを使用します。

6. Cloud Shell を起動し、`az account get-access-token` を実行して API テスト用の一時的な認証トークンを取得します。 以前にコピーしたエンドポイントとキーと共に保持します。

## 必要なモデルをデプロイする

Azure には、モデルのデプロイ、管理、調査に使用できる **Azure AI Studio** という名前の Web ベース ポータルが用意されています。 Azure AI Studio を使用してモデルをデプロイすることで、Azure OpenAI の調査を開始します。

> **注**: Azure AI Studio を使用すると、実行するタスクを提案するメッセージ ボックスが表示される場合があります。 これらを閉じて、この演習の手順に従うことができます。

1. Azure portal にある Azure OpenAI リソースの **[概要]** ページで、**[開始する]** セクションまで下にスクロールし、ボタンを選択して **Azure AI Studio** に移動します。
   
1. Azure AI Studio の左ペインで、**[デプロイ]** ページを選び、既存のモデル デプロイを表示します。 まだデプロイがない場合は、次の設定で **gpt-35-turbo** モデルの新しいデプロイを作成します。
    - **デプロイ名**: *gpt-35-turbo-0613*
    - **モデル**: gpt-35-turbo
    - **モデル バージョン**: 0613
    - **デプロイの種類**:Standard
    - **1 分あたりのトークンのレート制限**: 5K\*
    - **コンテンツ フィルター**: 既定
    - **動的クォータを有効にする**: 無効
    
> \* この演習は、1 分あたり 5,000 トークンのレート制限内で余裕を持って完了できます。またこの制限によって、同じサブスクリプションを使用する他のユーザーのために容量を残すこともできます。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

1. **Azure portal** (`https://portal.azure.com`) にサインインします。
2. 次の設定で **Azure Databricks** リソースを作成します。
    - **サブスクリプション**: *Azure OpenAI リソースの作成に使用したサブスクリプションと同じ Azure サブスクリプションを選択します*
    - **リソース グループ**: *Azure OpenAI リソースを作成したリソース グループと同じです*
    - **リージョン**: *Azure OpenAI リソースを作成したリージョンと同じです*
    - **[名前]**: "*希望する一意の名前*"
    - **価格レベル**: *Premium* または*試用版*

3. **[確認および作成]** を選択し、デプロイが完了するまで待ちます。 次にリソースに移動し、ワークスペースを起動します。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS **<u>ML</u>** 以降のランタイム バージョンを備えたクラスターが既にある場合は、この手順をスキップし、そのクラスターを使用してこの演習を完了できます。

1. Azure portal で、Azure Databricks ワークスペースが作成されたリソース グループを参照します。
2. Azure Databricks サービス リソースを選択します。
3. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

> **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

4. 左側のサイドバーで、**[(+) 新規]** タスクを選択し、**[クラスター]** を選択します。
5. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
    - **クラスター名**: "ユーザー名の" クラスター (既定のクラスター名)**
    - **ポリシー**:Unrestricted
    - **クラスター モード**: 単一ノード
    - **アクセス モード**: 単一ユーザー (*自分のユーザー アカウントを選択*)
    - **Databricks Runtime のバージョン**: "以下に該当する最新の非ベータ版ランタイム (標準ランタイム バージョン**ではない***) の **<u>ML</u>** エディションを選択します。"
        - "*GPU を使用**しない***"
        - *Scala > **2.11** を含める*
        - "**3.4** 以上の Spark を含む"**
    - **Photon Acceleration を使用する**: <u>オフ</u>にする
    - **ノード タイプ**: Standard_D4ds_v5
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

6. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。

## 必要なライブラリをインストールする

1. クラスターのページで、**[ライブラリ]** タブを選択します。

2. **[新規インストール]** を選択します。

3. ライブラリ ソースとして **PyPI** を選択し、次の Python パッケージをインストールします。
   - `numpy==2.1.0`
   - `requests==2.32.3`
   - `openai==1.42.0`
   - `tiktoken==0.7.0`

## 新しいノートブックの作成とデータの取り込み

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
   
1. ノートブックに名前を付け、**[接続]** ドロップダウン リストで、まだ選択されていない場合はクラスターを選択します。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

2. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使用して、GitHub からクラスターで使用されるファイル システムにデータ ファイルをダウンロードします。**

     ```python
    %sh
    rm -r /dbfs/fine_tuning
    mkdir /dbfs/fine_tuning
    wget -O /dbfs/fine_tuning/training_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/training_set.jsonl
    wget -O /dbfs/fine_tuning/validation_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/validation_set.jsonl
     ```

3. 新しいセルで、この演習の冒頭でコピーしたアクセス情報を含む次のコードを実行して、Azure OpenAI リソースを使用するときに認証用の永続的な環境変数を割り当てます。

     ```python
    import os

    os.environ["AZURE_OPENAI_API_KEY"] = "your_openai_api_key"
    os.environ["AZURE_OPENAI_ENDPOINT"] = "your_openai_endpoint"
    os.environ["TEMP_AUTH_TOKEN"] = "your_access_token"
     ```
     
## 有効性トークン数

`training_set.jsonl` と `validation_set.jsonl` はどちらも、微調整されたモデルをトレーニングおよび検証するためのデータ ポイントとして機能する、`user` と `assistant` の間でのさまざまな会話の例で構成されています。 個々の例は、`gpt-35-turbo` モデルの入力トークン制限である 4096 トークン以下に留める必要があります。

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

    files = ['/dbfs/fine_tuning/training_set.jsonl', '/dbfs/fine_tuning/validation_set.jsonl']

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

## 微調整ファイルを Azure OpenAI にアップロードする

モデルの微調整を開始する前に、OpenAI クライアントを初期化し、その環境に微調整ファイルを追加して、ジョブの初期化に使用されるファイル ID を生成する必要があります。

1. 新しいセルで次のコードを実行します。

     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      api_key = os.getenv("AZURE_OPENAI_API_KEY"),
      api_version = "2024-05-01-preview"  # This API version or later is required to access seed/events/checkpoint features
    )

    training_file_name = '/dbfs/fine_tuning/training_set.jsonl'
    validation_file_name = '/dbfs/fine_tuning/validation_set.jsonl'

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

微調整ファイルが正常にアップロードされたので、微調整トレーニング ジョブを送信できるようになりました。 トレーニングが完了するまでに 1 時間以上かかるのは珍しいことではありません。 トレーニングが完了したら、左側のウィンドウで **[微調整]** オプションを選択すると、Azure AI Studio で結果を確認できます。

1. 新しいセルで、次のコードを実行して、微調整トレーニング ジョブを開始します。

     ```python
    response = client.fine_tuning.jobs.create(
        training_file = training_file_id,
        validation_file = validation_file_id,
        model = "gpt-35-turbo-0613",
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

3. ジョブの状態が `succeeded` に変わったら、次のコードを実行して、最終的な結果を取得します。

     ```python
    response = client.fine_tuning.jobs.retrieve(job_id)

    print(response.model_dump_json(indent=2))
    fine_tuned_model = response.fine_tuned_model
     ```
   
## 微調整されたモデルをデプロイする

これで微調整されたモデルを取得できたので、カスタマイズしたモデルとしてデプロイし、Azure OpenAI Studio の**チャット** プレイグラウンド、またはチャット入力候補 API のいずれかを使用して、他のデプロイ済みモデルと同様に使用できます。

1. 新しいセルで、次のコードを実行して、微調整されたモデルをデプロイします。
   
     ```python
    import json
    import requests

    token = os.getenv("TEMP_AUTH_TOKEN")
    subscription = "<YOUR_SUBSCRIPTION_ID>"
    resource_group = "<YOUR_RESOURCE_GROUP_NAME>"
    resource_name = "<YOUR_AZURE_OPENAI_RESOURCE_NAME>"
    model_deployment_name = "gpt-35-turbo-ft"

    deploy_params = {'api-version': "2023-05-01"}
    deploy_headers = {'Authorization': 'Bearer {}'.format(token), 'Content-Type': 'application/json'}

    deploy_data = {
        "sku": {"name": "standard", "capacity": 1},
        "properties": {
            "model": {
                "format": "OpenAI",
                "name": "<YOUR_FINE_TUNED_MODEL>",
                "version": "1"
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
      api_key = os.getenv("AZURE_OPENAI_API_KEY"),
      api_version = "2024-02-01"
    )

    response = client.chat.completions.create(
        model = "gpt-35-turbo-ft", # model = "Custom deployment name you chose for your fine-tuning model"
        messages = [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Does Azure OpenAI support customer managed keys?"},
            {"role": "assistant", "content": "Yes, customer managed keys are supported by Azure OpenAI."},
            {"role": "user", "content": "Do other Azure AI services support this too?"}
        ]
    )

    print(response.choices[0].message.content)
     ```
 
## クリーンアップ

Azure OpenAI リソースでの作業が完了したら、**Azure portal** (`https://portal.azure.com`) でデプロイまたはリソース全体を忘れずに削除します。

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
