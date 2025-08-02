---
lab:
  title: Azure Databricks を使用した取得拡張生成
---

# Azure Databricks を使用した取得拡張生成

取得拡張生成 (RAG) は、外部のナレッジ ソースを統合することによって大規模言語モデルを強化する AI の最先端のアプローチです。 Azure Databricks は、RAG アプリケーションを開発するための堅牢なプラットフォームを提供します。これにより、非構造化データを取得と応答の生成に適した形式に変換できます。 このプロセスには、ユーザーのクエリの理解、関連するデータの取得、言語モデルを使用した応答の生成など、一連の手順が含まれます。 Azure Databricks によって提供されるフレームワークは、RAG アプリケーションの迅速な反復とデプロイをサポートし、最新の情報と独自の知識を含めることができる高品質のドメイン固有の応答を保証します。

このラボは完了するまで、約 **40** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。***PowerShell*** 環境を選択します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#10530;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

    ```powershell
   rm -r mslearn-databricks -f
   git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. リポジトリをクローンした後、次のコマンドを入力して **setup.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

    ```powershell
   ./mslearn-databricks/setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。

7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 16.4 LTS **<u>ML</u>** 以降のランタイム バージョンを備えたクラスターが既にある場合は、それを使ってこの演習を完了し、この手順をスキップできます。

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します
1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。
1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

1. 左側のサイドバーで、**[(+) 新規]** タスクを選択し、**[クラスター]** を選択します。
1. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
    - **クラスター名**: "ユーザー名の" クラスター (既定のクラスター名)**
    - **ポリシー**:Unrestricted
    - **機械学習**: 有効
    - **Databricks Runtime**:16.4 LTS
    - **Photon Acceleration を使用する**: <u>オフ</u>にする
    - **ワーカー タイプ**:Standard_D4ds_v5
    - **シングル ノード**:オン

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## 必要なライブラリをインストールする

1. クラスターのページで、**[ライブラリ]** タブを選択します。

2. **[新規インストール]** を選択します。

3. ライブラリ ソースとして **[PyPI]** を選択し、**"パッケージ"** フィールドに「`transformers==4.53.0`」と入力します。

4. **[インストール]** を選択します。

5. 上記の手順を繰り返して、`databricks-vectorsearch==0.56`もインストールします。
   
## ノートブックを作成してデータを取り込む

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。 **[接続]** ドロップダウン リストで、まだ選択されていない場合はクラスターを選択します。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

1. ノートブックの最初のセルで、次の SQL クエリを入力して、この演習のデータを既定のカタログ内に格納するために使用する新しいボリュームを作成します。

    ```python
   %sql 
   CREATE VOLUME <catalog_name>.default.RAG_lab;
    ```

1. `<catalog_name>` をワークスペースの名前に置き換えます。これは、Azure Databricks によって既定のカタログがその名前で自動的に作成されるためです。
1. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。
1. 新しいセルで、次のコードを実行して、"シェル" コマンドを使用して GitHub から Unity カタログにデータをダウンロードします。**

    ```python
   %sh
   wget -O /Volumes/<catalog_name>/default/RAG_lab/enwiki-latest-pages-articles.xml https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/enwiki-latest-pages-articles.xml
    ```

1. 新しいセルで、次のコードを実行して、生データからデータフレームを作成します。

    ```python
   from pyspark.sql import SparkSession

   # Create a Spark session
   spark = SparkSession.builder \
       .appName("RAG-DataPrep") \
       .getOrCreate()

   # Read the XML file
   raw_df = spark.read.format("xml") \
       .option("rowTag", "page") \
       .load("/Volumes/<catalog_name>/default/RAG_lab/enwiki-latest-pages-articles.xml")

   # Show the DataFrame
   raw_df.show(5)

   # Print the schema of the DataFrame
   raw_df.printSchema()
    ```

1. 新しいセルで、次のコードを実行し、`<catalog_name>` を Unity カタログの名前に置き換えて、データをクリーンアップして前処理し、関連するテキスト フィールドを抽出します。

    ```python
   from pyspark.sql.functions import col

   clean_df = raw_df.select(col("title"), col("revision.text._VALUE").alias("text"))
   clean_df = clean_df.na.drop()
   clean_df.write.format("delta").mode("overwrite").saveAsTable("<catalog_name>.default.wiki_pages")
   clean_df.show(5)
    ```

    **カタログ (Ctrl + Alt + C)** エクスプローラーを開いてペインを更新すると、既定の Unity カタログに Delta テーブルが作成されます。

## 埋め込みを生成し、ベクトル検索を実装する

Databricks の Mosaic AI ベクトル検索は、Azure Databricks プラットフォーム内に統合されたベクトル データベース ソリューションです。 Hierarchical Navigable Small World (HNSW) アルゴリズムを使用して、埋め込みのストレージと取得を最適化します。 これにより、効率的な最近隣検索が可能になり、そのハイブリッド キーワード類似性検索機能は、ベクトル ベースとキーワード ベースの検索手法を組み合わせることにより、より関連性の高い結果を提供します。

1. 新しいセルで、差分同期インデックスを作成する前に、次の SQL クエリを実行してソース テーブルのデータ フィードの変更機能を有効にします。

    ```python
   %sql
   ALTER TABLE <catalog_name>.default.wiki_pages SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
    ```

2. 新しいセルで、次のコードを実行して、ベクトル検索インデックスを作成します。

    ```python
   from databricks.vector_search.client import VectorSearchClient

   client = VectorSearchClient()

   client.create_endpoint(
       name="vector_search_endpoint",
       endpoint_type="STANDARD"
   )

   index = client.create_delta_sync_index(
     endpoint_name="vector_search_endpoint",
     source_table_name="<catalog_name>.default.wiki_pages",
     index_name="<catalog_name>.default.wiki_index",
     pipeline_type="TRIGGERED",
     primary_key="title",
     embedding_source_column="text",
     embedding_model_endpoint_name="databricks-gte-large-en"
    )
    ```
     
**カタログ (Ctrl + Alt + C)** エクスプローラーを開いてペインを更新すると、既定の Unity カタログにインデックスが作成されます。

> **注:** 次のコード セルを実行する前に、インデックスが正常に作成されたことを確認します。 これを行うには、[カタログ] ペインでインデックスを右クリックし、**[カタログ エクスプローラーで開く]** を選択します。 インデックスの状態が **[オンライン]** になるまで待ちます。

3. 新しいセルで、次のコードを実行して、クエリ ベクトルに基づいて関連するドキュメントを検索します。

    ```python
   results_dict=index.similarity_search(
       query_text="Anthropology fields",
       columns=["title", "text"],
       num_results=1
   )

   display(results_dict)
    ```

出力で、クエリ プロンプトに関連する対応する Wiki ページが見つかることを確認します。

## 取得したデータを使用してプロンプトを拡張する

これで、外部データ ソースからの追加のコンテキストを提供することで、大規模言語モデルの機能を強化できるようになりました。 そうすることで、モデルはより正確でコンテキストに関連する応答を生成できます。

1. 新しいセルで、次のコードを実行して、取得したデータをユーザーのクエリと組み合わせて、LLM のリッチ プロンプトを作成します。

    ```python
   # Convert the dictionary to a DataFrame
   results = spark.createDataFrame([results_dict['result']['data_array'][0]])

   from transformers import pipeline

   # Load the summarization model
   summarizer = pipeline("summarization", model="facebook/bart-large-cnn", framework="pt")

   # Extract the string values from the DataFrame column
   text_data = results.select("_2").rdd.flatMap(lambda x: x).collect()

   # Pass the extracted text data to the summarizer function
   summary = summarizer(text_data, max_length=512, min_length=100, do_sample=True)

   def augment_prompt(query_text):
       context = " ".join([item['summary_text'] for item in summary])
       return f"Query: {query_text}\nContext: {context}"

   prompt = augment_prompt("Explain the significance of Anthropology")
   print(prompt)
    ```

3. 新しいセルで、次のコードを実行して、LLM を使用して応答を生成します。

    ```python
   from transformers import GPT2LMHeadModel, GPT2Tokenizer

   tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
   model = GPT2LMHeadModel.from_pretrained("gpt2")

   inputs = tokenizer(prompt, return_tensors="pt")
   outputs = model.generate(
       inputs["input_ids"], 
       max_length=300, 
       num_return_sequences=1, 
       repetition_penalty=2.0, 
       top_k=50, 
       top_p=0.95, 
       temperature=0.7,
       do_sample=True
   )
   response = tokenizer.decode(outputs[0], skip_special_tokens=True)

   print(response)
    ```

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
